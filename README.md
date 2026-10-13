# Bank Ledger

A backend service for a simple banking system built around a **double-entry ledger**. Account balances are never stored directly. They are derived by summing immutable `CREDIT` and `DEBIT` ledger entries, so every movement of money can be traced and audited.

Built with **Node.js, Express 5, MongoDB (Mongoose)** and JWT authentication.

---

## Features

- **User authentication**: register, login and logout with JWT (stored in a cookie or sent as a `Bearer` token)
- **Token blacklisting**: logged-out tokens are blacklisted and automatically expire from the DB after 3 days
- **Accounts**: users can create multiple accounts (default currency `INR`) and check balances
- **Immutable ledger**: ledger entries cannot be updated or deleted once written
- **Derived balances**: balance = total credits − total debits, computed with a MongoDB aggregation
- **Atomic transfers**: debit, credit and transaction status are written inside a single MongoDB session/transaction
- **Idempotency**: every transfer requires a unique `idempotencyKey`, so retries never double-spend
- **System user**: a privileged user that can seed initial funds into accounts
- **Email notifications**: welcome and transaction emails via Gmail OAuth2 (Nodemailer)

---

## Project Structure

```
bank-ledger/
├── server.js                      # Entry point: loads env, connects DB, starts server
├── package.json
└── src/
    ├── app.js                     # Express app, middleware and route mounting
    ├── config/
    │   └── db.js                  # MongoDB connection
    ├── controllers/
    │   ├── auth.controller.js         # Register / login / logout
    │   ├── account.controller.js      # Create account, list accounts, get balance
    │   └── transaction.controller.js  # Transfers and initial funds
    ├── middleware/
    │   └── auth.middleware.js         # authMiddleware, authSystemUserMiddleware
    ├── models/
    │   ├── user.model.js
    │   ├── account.model.js           # Includes getBalance() aggregation
    │   ├── transaction.model.js
    │   ├── ledger.model.js            # Immutable entries
    │   └── blackList.model.js         # Revoked JWTs (TTL index)
    ├── routes/
    │   ├── auth.routes.js
    │   ├── account.routes.js
    │   └── transaction.routes.js
    └── services/
        └── email.service.js           # Nodemailer + Gmail OAuth2
```

---

## Getting Started

### Prerequisites

- [Node.js](https://nodejs.org/) 18+
- A MongoDB deployment running as a **replica set** (required for multi-document transactions). MongoDB Atlas works out of the box; for local use, start `mongod` with `--replSet` and initiate it.
- A Google Cloud project with Gmail OAuth2 credentials (only needed for email notifications)

### Installation

```bash
git clone https://github.com/sahil-arsss/bank-ledger.git
cd bank-ledger
npm install
```

### Environment Variables

Create a `.env` file in the project root (it is already git-ignored):

```env
# Database
MONGO_URI=mongodb+srv://<user>:<password>@<cluster>/<db>

# Auth
JWT_SECRET=your_long_random_secret

# Email (Gmail OAuth2)
EMAIL_USER=your.email@gmail.com
CLIENT_ID=your_google_client_id
CLIENT_SECRET=your_google_client_secret
REFRESH_TOKEN=your_google_refresh_token
```

### Run

```bash
npm run dev     # development, auto-restart with nodemon
npm start       # production
```

The server listens on **http://localhost:3000**. A health check is available at `GET /`.

---

## API Reference

Base URL: `http://localhost:3000`

Protected routes accept the JWT either from the `token` cookie (set automatically on login/register) or an `Authorization: Bearer <token>` header.

### Auth

| Method | Endpoint             | Auth | Description                          |
| ------ | -------------------- | ---- | ------------------------------------ |
| POST   | `/api/auth/register` | No   | Create a user and receive a JWT      |
| POST   | `/api/auth/login`    | No   | Log in and receive a JWT             |
| POST   | `/api/auth/logout`   | No   | Blacklist the current token          |

**Register** body:

```json
{ "name": "Jane Doe", "email": "jane@example.com", "password": "secret123" }
```

**Login** body:

```json
{ "email": "jane@example.com", "password": "secret123" }
```

### Accounts

| Method | Endpoint                          | Auth | Description                                |
| ------ | --------------------------------- | ---- | ------------------------------------------ |
| POST   | `/api/accounts`                   | Yes  | Create a new account for the logged-in user |
| GET    | `/api/accounts`                   | Yes  | List the logged-in user's accounts         |
| GET    | `/api/accounts/balance/:accountId`| Yes  | Get the derived balance of one of your accounts |

### Transactions

| Method | Endpoint                               | Auth        | Description                                  |
| ------ | -------------------------------------- | ----------- | -------------------------------------------- |
| POST   | `/api/transactions`                    | Yes         | Transfer money between two accounts          |
| POST   | `/api/transactions/system/initial-funds` | System user | Credit an account from the system account    |

**Transfer** body:

```json
{
  "fromAccount": "<accountId>",
  "toAccount": "<accountId>",
  "amount": 500,
  "idempotencyKey": "a-unique-key-per-transfer"
}
```

**Initial funds** body:

```json
{
  "toAccount": "<accountId>",
  "amount": 10000,
  "idempotencyKey": "a-unique-key"
}
```

---

## How Transfers Work

`POST /api/transactions` follows a 10-step flow:

1. Validate the request body
2. Check the idempotency key: return the existing result if the transfer was already processed
3. Verify both accounts are `ACTIVE`
4. Derive the sender's balance from the ledger and check for sufficient funds
5. Create the transaction as `PENDING`
6. Write a `DEBIT` ledger entry for the sender
7. Write a `CREDIT` ledger entry for the receiver
8. Mark the transaction `COMPLETED`
9. Commit the MongoDB session (all of the above succeed or none do)
10. Send an email notification

### Data model at a glance

| Model         | Purpose                                                                   |
| ------------- | ------------------------------------------------------------------------- |
| `user`        | Name, unique email, hashed password (bcrypt), `systemUser` flag           |
| `account`     | Belongs to a user. Status: `ACTIVE`, `FROZEN`, `CLOSED`. Default `INR`    |
| `transaction` | `fromAccount`, `toAccount`, `amount`, `idempotencyKey`. Status: `PENDING`, `COMPLETED`, `FAILED`, `REVERSED` |
| `ledger`      | Immutable `CREDIT`/`DEBIT` entry linked to an account and a transaction   |
| `tokenBlackList` | Revoked JWTs, auto-deleted after 3 days                                |

---

## Setting Up a System User

The `systemUser` flag is **immutable** and not settable through the API. To create one:

1. Register a normal user via `/api/auth/register`
2. In MongoDB, create a user document with `systemUser: true` (or set it directly on a freshly created user before the flag is ever written elsewhere)
3. Create an account for that user via `POST /api/accounts`
4. Log in as the system user and call `/api/transactions/system/initial-funds` to fund other accounts

Note that `initial-funds` uses the system user's first account as the source and does not check its balance.

---

## Current Limitations / Notes

- **Artificial delay**: `createTransaction` contains a 15-second `setTimeout` between the debit and credit steps. This looks like a testing aid for the `PENDING` state and should be removed for real use.
- **Replica set required**: transfers use MongoDB sessions and will fail on a standalone MongoDB instance.
- **No tests yet**: `npm test` is a placeholder.
- **Ownership check**: transfers currently verify that both accounts exist, but do not verify that `fromAccount` belongs to the logged-in user.
- **Cookie settings**: the auth cookie is set without `httpOnly` / `secure` options.
- The port (`3000`) is hard-coded in `server.js`.

---

## Tech Stack

- [Express 5](https://expressjs.com/)
- [Mongoose 9](https://mongoosejs.com/) / MongoDB
- [jsonwebtoken](https://github.com/auth0/node-jsonwebtoken) and [bcryptjs](https://github.com/dcodeIO/bcrypt.js)
- [Nodemailer](https://nodemailer.com/) (Gmail OAuth2)
- [cookie-parser](https://github.com/expressjs/cookie-parser), [dotenv](https://github.com/motdotla/dotenv)

## License

ISC
