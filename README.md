# ChainVault

ChainVault is a decentralized file storage and sharing platform that combines blockchain technology for immutable audit trails with IPFS for distributed file storage. It provides secure, encrypted file uploads with granular access control, all backed by a proof-of-work blockchain for tamper-proof records.

## Table of Contents

- [Overview](#overview)
- [How It Works](#how-it-works)
- [Blockchain Implementation](#blockchain-implementation)
- [IPFS Integration](#ipfs-integration)
- [Encryption](#encryption)
- [Authentication and Access Control](#authentication-and-access-control)
- [Architecture](#architecture)
- [Setup and Installation](#setup-and-installation)
- [Usage](#usage)
- [API Endpoints](#api-endpoints)
- [Deployment](#deployment)
- [Contributing](#contributing)

## Overview

ChainVault allows users to upload files that are:
- **Encrypted** using Fernet (AES-128) symmetric encryption
- **Stored** on IPFS (InterPlanetary File System) for decentralization
- **Recorded** on a custom blockchain for immutable audit trails
- **Controlled** with role-based access management

The system maintains a complete history of file operations (uploads, access grants, revokes) in an immutable blockchain, while files themselves are stored encrypted on IPFS.

## How It Works

1. **User Registration/Login**: Users create accounts with username/password. The first user becomes an admin.

2. **File Upload**:
   - User selects a file and uploads it
   - File is hashed (SHA-256) for integrity verification
   - File is encrypted using Fernet encryption
   - Encrypted file is uploaded to IPFS, receiving a Content Identifier (CID)
   - A blockchain record is created containing file metadata, CID, hash, owner, and access list

3. **Access Control**:
   - File owner can grant/revoke access to other users
   - Each access change creates a new blockchain record
   - Only authorized users can download or view file details

4. **File Download**:
   - User requests file by ID
   - System checks access permissions
   - Retrieves encrypted file from IPFS using CID
   - Decrypts file and serves it to user

5. **Verification**:
   - File integrity can be verified by comparing stored hash with computed hash
   - Blockchain validity can be checked by re-computing all hashes

## Blockchain Implementation

ChainVault implements a custom proof-of-work blockchain similar to Bitcoin but simplified for file record management.

### Block Structure

Each block contains:
- `index`: Block number (sequential)
- `timestamp`: Unix timestamp (integer)
- `data`: Record data (file upload, access change, etc.)
- `previous_hash`: Hash of previous block
- `nonce`: Proof-of-work nonce
- `hash`: SHA-256 hash of block contents

### Proof-of-Work

- Difficulty: 3 leading zeros required
- Mining: Increment nonce until hash starts with "000"
- Computation: SHA-256 of JSON-serialized block data

### Record Types

The blockchain stores different types of records:

1. **GENESIS**: Initial block
2. **FILE_UPLOAD**: File upload with metadata
3. **ACCESS_GRANT**: Adding user to access list
4. **ACCESS_REVOKE**: Removing user from access list

### Validation

The blockchain is validated by:
- Re-computing each block's hash
- Verifying hash matches stored hash
- Checking previous_hash links are correct
- Ensuring proof-of-work difficulty is met

### Persistence

Blocks are stored in MongoDB Atlas for persistence across restarts. The system loads existing blocks on startup and continues the chain.

## IPFS Integration

IPFS (InterPlanetary File System) provides decentralized, content-addressed storage.

### How IPFS Works in ChainVault

1. **Upload**:
   - Encrypted file bytes are sent to Pinata IPFS pinning service
   - Receives a Content Identifier (CID) - a unique hash of the content
   - CID is stored in blockchain record

2. **Download**:
   - CID retrieved from blockchain
   - File downloaded from IPFS gateway (gateway.pinata.cloud)
   - Content addressing ensures file integrity

3. **Fallback**:
   - If Pinata credentials not configured, falls back to local file storage
   - Local files stored with SHA-256 hash as filename

### Benefits of IPFS

- **Decentralized**: No single point of failure
- **Content Addressing**: Files identified by content hash, not location
- **Immutable**: Same content always produces same CID
- **Distributed**: Files can be served from multiple nodes

## Encryption

ChainVault uses Fernet symmetric encryption for file security.

### Fernet Encryption

- **Algorithm**: AES-128 in CBC mode with PKCS7 padding
- **Authentication**: HMAC-SHA256 for integrity
- **Key Derivation**: PBKDF2 with 100,000 iterations from passphrase
- **Salt**: Fixed salt for deterministic key generation

### Key Points

- Same passphrase always generates same encryption key
- Encryption includes authentication tag to prevent tampering
- Files are encrypted before IPFS upload, decrypted after download
- Encryption/decryption is transparent to users

## Authentication and Access Control

### User Management

- **Registration**: Username/password with bcrypt hashing
- **Roles**: "admin" (first user) or "user"
- **Session**: Flask session-based authentication

### Access Control

- **Ownership**: File owner has full control
- **Access List**: Array of usernames with access
- **Admin Override**: Admins can access any file
- **Audit Trail**: All access changes recorded on blockchain

## Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Web Frontend  │    │   Flask API     │    │   MongoDB Atlas │
│   (HTML/JS)     │◄──►│   (Python)      │◄──►│   (Blocks/Users) │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                                │
                                ▼
                       ┌─────────────────┐    ┌─────────────────┐
                       │   IPFS/Pinata   │    │   Blockchain     │
                       │   (File Store)  │    │   (Audit Trail)  │
                       └─────────────────┘    └─────────────────┘
```

### Components

- **Frontend**: Vanilla JavaScript web interface
- **API**: Flask REST API with CORS support
- **Blockchain**: Custom Python implementation
- **Storage**: IPFS for files, MongoDB for metadata
- **Auth**: Session-based with bcrypt passwords

## Setup and Installation

### Prerequisites

- Python 3.8+
- MongoDB Atlas account (free tier available)
- Pinata IPFS account (optional, falls back to local storage)

### Local Development

1. **Clone the repository**
   ```bash
   git clone <repository-url>
   cd chainvault
   ```

2. **Create virtual environment**
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

4. **Environment variables**
   Create a `.env` file:
   ```env
   SECRET_KEY=your-secret-key-here
   MONGO_URI=mongodb+srv://username:password@cluster.mongodb.net/
   ENCRYPTION_PASSPHRASE=your-encryption-passphrase
   PINATA_KEY=your-pinata-api-key  # Optional
   PINATA_SECRET=your-pinata-secret  # Optional
   ```

5. **Run the application**
   ```bash
   python api/server.py
   ```

   Or with gunicorn:
   ```bash
   gunicorn api.server:app --timeout 120
   ```

6. **Access the application**
   Open http://localhost:5000 in your browser

### First User

The first user to register becomes an admin. Visit `/register` to create your account.

## Usage

### Web Interface

1. **Register/Login**: Create account or login
2. **Upload Files**: Use the Upload tab to select and upload files
3. **Manage Access**: Grant/revoke access to other users
4. **Download Files**: Download files you have access to
5. **Verify Integrity**: Check file hashes against blockchain records
6. **View History**: See complete audit trail for any file

### File Operations

- **Upload**: Files are encrypted and stored on IPFS, metadata on blockchain
- **Access Control**: Owner can share with specific users
- **Verification**: Compare file hash with blockchain record
- **History**: View all operations performed on a file

## API Endpoints

### Authentication
- `POST /api/auth/register` - Register new user
- `POST /api/auth/login` - Login user
- `POST /api/auth/logout` - Logout user
- `GET /api/auth/me` - Get current user info

### File Operations
- `POST /api/upload` - Upload file
- `GET /api/file/<file_id>` - Get file metadata
- `GET /api/download/<file_id>` - Download file
- `POST /api/grant` - Grant access to user
- `POST /api/revoke` - Revoke access from user
- `GET /api/verify/<file_id>/<hash>` - Verify file integrity
- `GET /api/history/<file_id>` - Get file history
- `GET /api/files` - Get all accessible files

### Blockchain
- `GET /api/chain` - Get blockchain info
- `POST /api/admin/reset-chain` - Reset blockchain (admin only)

## Deployment

### Render

1. Create a new Web Service on Render
2. Connect your GitHub repository
3. Use the following settings:
   - **Runtime**: Python
   - **Build Command**: `pip install -r requirements.txt`
   - **Start Command**: `gunicorn api.server:app --timeout 120 --workers 1`
4. Set environment variables in Render dashboard:
   - `MONGO_URI`
   - `SECRET_KEY`
   - `ENCRYPTION_PASSPHRASE`
   - `PINATA_KEY` (optional)
   - `PINATA_SECRET` (optional)

### Heroku

Use the provided `Procfile` for Heroku deployment.

### Local Production

For production use, set `debug=False` and use a proper WSGI server like gunicorn.

## Contributing

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Test thoroughly
5. Submit a pull request

### Development Notes

- The blockchain implementation is simplified for educational purposes
- In production, consider using established blockchain platforms
- IPFS integration can be extended to use local IPFS nodes
- Encryption uses symmetric keys - consider asymmetric for enhanced security

## Security Considerations

- **Encryption**: Files are encrypted at rest and in transit
- **Access Control**: Granular permissions with blockchain audit trail
- **Authentication**: Secure password hashing with bcrypt
- **Blockchain**: Immutable records prevent tampering
- **IPFS**: Decentralized storage reduces single points of failure

## License

This project is open source. Please check the license file for details.