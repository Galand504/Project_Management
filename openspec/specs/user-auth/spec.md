# User Authentication Specification

## Purpose

Manages user identity lifecycle: registration, login, logout, and session security. Ensures only authenticated users access protected resources and that sessions resist hijacking and CSRF attacks.

## Requirements

### Requirement: User Registration

The system SHALL allow new users to create an account with email and password.

#### Scenario: Successful registration

- GIVEN a visitor on the registration page
- WHEN they submit a valid email, password (≥8 chars, 1 uppercase, 1 digit), and confirm password
- THEN the system creates the account, hashes the password, and redirects to login

#### Scenario: Duplicate email

- GIVEN an email already registered
- WHEN a visitor submits that email during registration
- THEN the system rejects the submission and shows "Email already in use"

#### Scenario: Password mismatch

- GIVEN the user enters mismatched password and confirm-password fields
- WHEN they submit the form
- THEN the system rejects the submission and shows "Passwords do not match"

### Requirement: User Login

The system SHALL authenticate users via email and password and establish a session.

#### Scenario: Successful login

- GIVEN a registered user on the login page
- WHEN they submit correct email and password
- THEN the system creates a session, sets a session cookie, and redirects to the dashboard

#### Scenario: Invalid credentials

- GIVEN a user on the login page
- WHEN they submit an incorrect email or password
- THEN the system rejects the attempt and shows "Invalid email or password"

#### Scenario: Brute-force protection

- GIVEN a user with 5 consecutive failed login attempts
- WHEN they attempt a 6th login
- THEN the system locks the account for 15 minutes and notifies the user

### Requirement: User Logout

The system SHALL destroy the user session on logout.

#### Scenario: Successful logout

- GIVEN an authenticated user
- WHEN they click logout
- THEN the system destroys the session, invalidates the cookie, and redirects to login

### Requirement: Session Security

The system SHALL protect sessions against hijacking and fixation.

#### Scenario: Session regeneration on login

- GIVEN a user with a session ID before login
- WHEN they successfully authenticate
- THEN the system issues a new session ID

#### Scenario: Session expiry

- GIVEN an authenticated user inactive for 30 minutes
- WHEN they request a protected page
- THEN the system destroys the session and redirects to login

#### Scenario: Concurrent session limit

- GIVEN a user logged in on device A
- WHEN they log in on device B
- THEN the system allows the new session and MAY invalidate the old one

### Requirement: CSRF Protection

The system SHALL require a valid CSRF token on every state-changing request.

#### Scenario: Valid token accepted

- GIVEN an authenticated user on a form page
- WHEN they submit the form with a valid CSRF token
- THEN the system processes the request normally

#### Scenario: Missing or invalid token rejected

- GIVEN a request without a valid CSRF token
- WHEN the request reaches the server
- THEN the system returns HTTP 403 and does not process the change

### Requirement: Password Storage

The system SHALL store passwords using bcrypt with a cost factor ≥ 12.

#### Scenario: Hash verification

- GIVEN a user registers with a plaintext password
- WHEN the password is stored
- THEN the database contains a bcrypt hash, not the plaintext

#### Scenario: Legacy migration

- GIVEN a user with an MD5-hashed password from the old system
- WHEN they log in successfully
- THEN the system re-hashes the password with bcrypt and updates the stored hash
