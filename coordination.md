# Cross-Project Coordination Log

## Open Problems
- [ ] Client cannot deserialize MODULE_BUNDLE
- [ ] Backend unsure when to resend module state

## Decisions
- Backend sends MODULE_BUNDLE only after AUTH_OK
- Client treats checksum mismatch as fatal

## Requests
- Client → Backend: clarify checksum algorithm