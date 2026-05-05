## Google OAuth Integration for Backend

This guide explains how to set up and test Google OAuth on the backend.

### ✅ Status

The backend already has:
- ✅ `/auth/google-login` endpoint that accepts Google ID tokens
- ✅ Google token verification using `google-auth` library
- ✅ Automatic user role detection (Student/Faculty)
- ✅ JWT token generation for authenticated sessions

You just need to:
1. Get a **Google Web Client ID** from Google Cloud Console
2. Set it as an environment variable on your backend

### 🔑 Configuration

The backend reads `GOOGLE_CLIENT_ID` from environment variables (see `app/config.py`):

```python
GOOGLE_CLIENT_ID: str | None = None
```

#### For Local Development:

Create a `.env` file in the `universe-backend/` directory:

```bash
GOOGLE_CLIENT_ID=YOUR_WEB_CLIENT_ID.apps.googleusercontent.com
SECRET_KEY=your-super-secret-key-here
STUDENT_DATABASE_URL=postgresql://user:pass@localhost/student_db
MASTER_DATABASE_URL=postgresql://user:pass@localhost/master_db
FACULTY_DATABASE_URL=postgresql://user:pass@localhost/faculty_db
```

#### For Production (Render):

1. Go to your Render dashboard
2. Select your backend service
3. Go to **Environment** → **Add Environment Variable**
4. Add:
   ```
   GOOGLE_CLIENT_ID=YOUR_WEB_CLIENT_ID.apps.googleusercontent.com
   ```

### 🧪 Testing the Endpoint

**Request:**
```bash
curl -X POST http://localhost:8000/auth/google-login \
  -H "Content-Type: application/json" \
  -d '{"id_token": "YOUR_GOOGLE_ID_TOKEN"}'
```

**Success Response (200):**
```json
{
  "access_token": "eyJhbGc...",
  "refresh_token": "eyJhbGc...",
  "user": {
    "id": 123,
    "email": "user@university.edu",
    "role": "student",
    "name": "John Doe"
  }
}
```

**Error Responses:**

- **400**: Invalid or expired ID token
  ```json
  {"detail": "Invalid ID token"}
  ```

- **401**: GOOGLE_CLIENT_ID not configured
  ```json
  {"detail": "GOOGLE_CLIENT_ID is not configured"}
  ```

- **404**: User not found in student/faculty databases
  ```json
  {"detail": "User not found in student or faculty records"}
  ```

### 📚 How It Works

1. **Mobile app** sends Google ID token to `/auth/google-login`
2. **Backend** verifies the token using Google's public keys
3. **Backend** extracts email from the token
4. **Backend** checks if user exists in student/faculty databases
5. **Backend** auto-detects role:
   - If in faculty DB → role = `"faculty"`
   - If in student master DB → role = `"student"`
6. **Backend** generates JWT tokens and returns to mobile app
7. **Mobile app** stores tokens and uses them for subsequent API calls

### 🔍 Verification Logic

See `app/auth/utils.py` for the implementation:

```python
def verify_google_token(token: str) -> dict:
    """
    Verifies a Google ID token using Google's public keys.
    Returns decoded token payload with claims like:
      - iss (issuer)
      - sub (subject/user ID)
      - email
      - email_verified
      - name
    """
```

### 🔐 Security Considerations

- ID tokens are verified using Google's public certificates
- Only the `email` field is used from the token
- Tokens expire after 1 hour (configured in `ACCESS_TOKEN_EXPIRE_MINUTES`)
- Refresh tokens allow getting new access tokens without re-authenticating
- All tokens are JWT-signed with your `SECRET_KEY`

### 🚀 Next Steps

1. **Get Web Client ID** from [Google Cloud Console](https://console.cloud.google.com/)
2. **Set GOOGLE_CLIENT_ID** in your environment (local `.env` or Render)
3. **Test the endpoint** using the curl command above
4. **Mobile app** will automatically use it once configured

### 📞 Support

If you get "Invalid ID token" errors:
1. Verify the `GOOGLE_CLIENT_ID` matches the one in your Google Cloud project
2. Check that the ID token hasn't expired (tokens expire quickly)
3. Ensure the user's email exists in your student/faculty databases

If you get "User not found" errors:
1. The email from Google doesn't exist in your databases
2. Check your master student or faculty database
3. Make sure email addresses match exactly (case-insensitive)
