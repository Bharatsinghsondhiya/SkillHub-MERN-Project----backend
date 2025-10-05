# SkillHub - Backend (MERN)

This repository is the backend for the SkillHub MERN project. It provides REST endpoints for user and contact operations, and a Socket.IO based real-time chat and video-call signaling service.

## Contents

- `app.js` - Server entrypoint, Socket.IO setup and HTTP server boot.
- `routes/` - Express routes for users, chats and contact form.
- `controller/` - Request handlers and business logic.
- `models/` - Mongoose schemas for User, Contact, ChatRoom and Message.
- `middleware/auth.js` - Simple JWT cookie-based authentication middleware.

## Quick start

Requirements

- Node.js (recommend v16+ or v18+ for ESM support)
- npm
- A running MongoDB instance (local or Atlas)

1. Install dependencies

```powershell
cd C:\Users\bhara\Downloads\SkillHub-MERN-Project----backend-master
npm install
```

2. Create a `.env` file in the project root and set the environment variables (see `.env.example` below)

3. Start the server

```powershell
npm start
```

The server listens on `process.env.PORT` or `5000` by default and exposes both HTTP APIs and a Socket.IO server.

## .env example

Create a `.env` file with the following keys (do NOT commit your real secrets):

```
DB_URL=mongodb+srv://<user>:<password>@cluster0.mongodb.net/skillhub?retryWrites=true&w=majority
PORT=5000
TOKEN_SECRET=your_jwt_secret_here
EMAIL=your_gmail_address@example.com
EMAIL_PASSWORD=your_app_password_or_smtp_password
```

- `DB_URL` - MongoDB connection string (required)
- `TOKEN_SECRET` - JWT secret used by authentication (required for auth endpoints)
- `EMAIL` and `EMAIL_PASSWORD` - used by Nodemailer for sending verification / contact emails

## Running in development

- You can install `nodemon` globally or use the included dev dependency and run with `nodemon app.js` for auto-restarts.

## API endpoints

Base URL: `http://localhost:5000` (unless `PORT` env overrides it)

User

- POST `/user` - Create a new user
  - Body: { name, email, password, confirmPassword, contact }
- POST `/user/login` - Log in
  - Body: { email, password }
  - On success sets an HTTP-only cookie `token` with the JWT
- POST `/user/forgetPassword` - Reset password
  - Body: { email, newPassword, confirmPassword }
- POST `/user/verification` - Verify account (used by emailed form)
  - Body: { email }
- DELETE `/user/logout` - Clear the `token` cookie (requires auth middleware in routes)
- PUT `/user/profile/:userId` - Update user profile (requires auth middleware in routes)
  - Body may contain: name, contact, email, bio, country, city, skillToTeach, skillToLearn, profileImage
- DELETE `/user/delete/:userId` - Delete profile
- GET `/user/getAllUser` - Returns list of users (protected route, requires cookie based JWT auth)

Contact

- POST `/contact/sendquery` - Submit the contact form
  - Body: { firstName, lastName, email, message, agreedToPrivacyPolicy }

Chat (REST)

- POST `/chat/send-message` - Send a message (creates/saves message)
  - Body: { sender, receiver, message, roomId }
- POST `/chat/createRoom/:userId` - Create or get a chat room for the current user with `userId`
- GET `/chat/room/:roomId/messages` - Get messages for a room
  - Query params: `page` (default 1), `limit` (default 50)

Health

- GET `/health` - Simple health check that returns server status and connected users count

Notes about auth: the project uses a cookie named `token` signed with `TOKEN_SECRET`. The `auth` middleware reads `request.cookies.token` and verifies it.

## Socket.IO events

The server registers a number of socket events to support real-time chat and WebRTC signaling for video calls.

Client -> Server events

- `joinRoom` - payload: { roomId, userId }
  - Joins the socket to the provided `roomId`, creates chat room in DB if needed and returns `roomJoined` + `chatHistory` events.
- `sendMessage` - payload: { sender, receiver, message, senderName }
  - Saves the message and broadcasts `receiveMessage` to the chat room.
- `startVideoCall` - payload: { to, from, fromName, offer, roomId }
- `acceptVideoCall` - payload: { to, from, answer, roomId }
- `declineVideoCall` - payload: { to, from, roomId }
- `endVideoCall` - payload: { to, from, roomId }
- `iceCandidate` - payload: { candidate, to, roomId }
- `ping` - health ping (server emits `pong`)

Server -> Client events

- `roomJoined` - confirmation after joining a room
- `chatHistory` - array of past messages for the room
- `receiveMessage` - emitted to room when a new message is saved
- `messageSent` - acknowledgement to sender that message was saved
- `messageError` - error saving message
- `userJoinedRoom` - notification sent to others when a user joins
- `incomingVideoCall` - signaling offer for incoming call
- `videoCallAccepted`, `videoCallDeclined`, `videoCallEnded` - call lifecycle events
- `iceCandidate` - relays ICE candidates during WebRTC handshake
- `userOffline` - notification a user is offline

Socket notes

- The server maintains an in-memory `connectedUsers` Map mapping userId -> socketId. This is used to forward events to specific users.
- Rooms are identified using a `roomId` string formed by joining two user ids with `_` (e.g. `id1_id2`). ChatRoom records in DB also use `roomId`.

## Models (summary)

User (`models/user.model.js`)
- name: String (required)
- email: String (required)
- password: String (required, hashed via bcrypt setter)
- contact: Number
- bio, country, city, skillToTeach, skillToLearn, profileImage
- isVerified: Boolean (default false)

Contact (`models/Contact.model.js`)
- firstName, lastName, email, message, agreedToPrivacyPolicy

ChatRoom (`models/chatRoom.model.js`)
- participants: [ObjectId ref user]
- (timestamps)

Message (`models/chatMessage.model.js`)
- chatRoom: ObjectId ref Chat
- sender: ObjectId ref user
- receiver: ObjectId ref user
- message: String
- senderName: String
- (timestamps)

## Troubleshooting & tips

- If you see `Missing required environment variable: DB_URL` on startup, make sure `.env` exists and `DB_URL` is set.
- If emails fail to send using Gmail, you may need to use an App Password (for accounts with 2FA) or configure SMTP properly.
- The `auth` middleware expects the JWT to be present in an HTTP-only cookie named `token`. If testing with tools like Postman either set that cookie or adapt the middleware.

## Suggested next steps / Improvements

- Add comprehensive request/response validation and clearer error codes.
- Add unit / integration tests for controllers.
- Persist online user state in a shared store (Redis) for multi-instance Socket.IO scaling.
- Add rate-limiting and input sanitization for public endpoints.

## License

This project currently does not include a license file. Add one as needed.

---

If you'd like, I can also:
- add an `.env.example` file to the repo,
- add Postman / HTTPie examples for each endpoint,
- or generate a small Postman collection for manual testing.
