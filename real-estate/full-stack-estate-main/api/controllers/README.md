# AuthController Documentation

This documentation provides an overview of the `AuthController` implementation, which includes user authentication using JWT, cookie management, and database interaction with Prisma and MongoDB.
![Screenshot 2024-06-03 234451](https://github.com/aryanvyas16/Real-Estate/assets/113963972/e403d7e7-bd8c-4a15-a238-92346f02a01e)


## Overview

The `AuthController` handles user authentication by hashing passwords, generating JWT tokens, and managing user sessions with cookies. The following sections will guide you through the process.

## Dependencies

- **Postman**: To test our API endpoints.
- **Bcrypt**: For password hashing.
- **Prisma**: For database interaction.
- **MongoDB**: As our database provider.
- **Cookie Parser**: To parse and manage cookies.
- **JWT**: For generating and verifying tokens.

## Implementation Details

### Password Hashing

We use bcrypt to hash passwords before saving them to our database. This ensures that even if the database is compromised, the passwords remain secure.

### Database Interaction

Prisma is used to interact with our MongoDB database. We save user details such as username, email, and hashed password to the database.

### Cookie Management

We use cookies to store user sessions. Cookies are set with a name and value, and managed using the `cookie-parser` library to simplify JWT handling.
![Screenshot 2024-06-06 162718](https://github.com/aryanvyas16/Real-Estate/assets/113963972/4d967e12-348c-4568-a7f5-5c4c56b0534a)


### JWT Token

- **Token Generation**: We hash the user ID and include it in the JWT token.
- **Token Usage**: When a user makes a request, we decrypt the token to retrieve the user ID.
- **Token Expiry**: Cookies are set to expire in one week.

### User Requests

- **Authentication**: After logging in, user information is stored in a cookie.
- **Authorization**: When a user makes a request, the cookie is sent to the API. The API decrypts the cookie to verify the user’s identity and permissions.

### Error Handling

- **Invalid Token**: If a user makes a request with an invalid token, an error is returned.
- **Unauthorized Actions**: If a user tries to delete a post that does not belong to them, an error is returned.
- ![Screenshot 2024-06-07 230438](https://github.com/aryanvyas16/Real-Estate/assets/113963972/993c5c75-a832-4c78-8634-33d93680ed55)


### Example Workflow

1. **User Registration**: Hash the user's password and save the user details to the database.
2. **User Login**: Generate a JWT token, store it in a cookie, and send the cookie to the client.
3. **Authenticated Requests**: Include the cookie in requests to the API. The API decrypts the cookie to verify the user.
4. **Authorization Check**: Before performing actions like deleting a post, check if the user ID from the token matches the post owner.

## Code Example

```javascript
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const cookieParser = require('cookie-parser');
const prisma = require('@prisma/client');

const registerUser = async (req, res) => {
  const { username, email, password } = req.body;
  const hashedPassword = await bcrypt.hash(password, 10);
  
  const newUser = await prisma.user.create({
    data: {
      username,
      email,
      password: hashedPassword,
    },
  });

  res.status(201).json(newUser);
};

const loginUser = async (req, res) => {
  const { email, password } = req.body;
  const user = await prisma.user.findUnique({ where: { email } });

  if (!user || !await bcrypt.compare(password, user.password)) {
    return res.status(401).json({ error: 'Invalid credentials' });
  }

  const token = jwt.sign({ userId: user.id }, 'your-secret-key', { expiresIn: '1w' });
  res.cookie('authToken', token, { httpOnly: true, maxAge: 7 * 24 * 60 * 60 * 1000 }); // 1 week
  res.status(200).json({ message: 'Login successful' });
};

const authenticate = (req, res, next) => {
  const token = req.cookies.authToken;
  if (!token) {
    return res.status(401).json({ error: 'Unauthorized' });
  }

  jwt.verify(token, 'your-secret-key', (err, decoded) => {
    if (err) {
      return res.status(401).json({ error: 'Unauthorized' });
    }
    req.userId = decoded.userId;
    next();
  });
};

// Example of an authorized action
const deletePost = async (req, res) => {
  const { postId } = req.params;
  const post = await prisma.post.findUnique({ where: { id: postId } });

  if (!post || post.userId !== req.userId) {
    return res.status(403).json({ error: 'Forbidden' });
  }

  await prisma.post.delete({ where: { id: postId } });
  res.status(200).json({ message: 'Post deleted' });
};

module.exports = {
  registerUser,
  loginUser,
  authenticate,
  deletePost,
};
