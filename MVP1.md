# Family Cloud — MVP 1

## Goal

Create a reliable local family cloud running on one Windows 11 computer.

This MVP replaces the basic functionality of Google Drive and Google Photos.

## Services

- Nextcloud for documents and general files
- Immich for photos, videos and automatic phone backup
- Docker Compose for installation and management

## Required features

- Separate accounts for family members
- Admin account
- Upload and download documents
- Automatic phone photo and video backup
- Private files for every user
- Shared family folders and albums
- Access through the home network
- Persistent storage after container restarts
- Clear setup and recovery documentation

## Not included yet

- Public internet exposure
- Router port forwarding
- Tailscale
- Advanced AI video scene search
- Document RAG
- Custom frontend or mobile application
- Production clustering
- Complex monitoring

## Technical requirements

- Must run using Docker Compose
- Must support Windows 11 with Docker Desktop and WSL2
- Use `.env` for secrets
- Provide `.env.example`
- Use persistent named volumes or clearly documented bind mounts
- Add health checks where practical
- Do not commit passwords or secrets
- Do not expose database services publicly
- Do not delete existing files automatically
- Keep Nextcloud and Immich media storage separate

## Deliverables

1. `docker-compose.yml`
2. `.env.example`
3. Storage folder structure
4. Installation instructions
5. First-admin setup instructions
6. Phone backup testing instructions
7. Local network access instructions
8. Backup and restore notes
9. Troubleshooting section

## Done when

- `docker compose config` succeeds
- All containers start successfully
- Nextcloud opens from another device on the LAN
- Immich opens from a phone on the LAN
- A Nextcloud user can upload and download a document
- Immich can upload a phone photo
- Data remains after restarting the containers