# PokerHole

> Texas Hold'em Poker Game - Server and Client

---

## Overview

PokerHole is a multiplayer Texas Hold'em poker game with a modern Spring Boot backend and terminal-based client.

This repository is a parent project containing two submodules:

- **[pokerhole-server](./pokerhole-server)** - Spring Boot 4.x + Java 21 backend server
- **[pokerhole-cli](./pokerhole-cli)** - Go + Bubble Tea terminal client

---

## Quick Start

### Clone with Submodules

```bash
git clone --recursive https://github.com/bunnyholes/pokerhole.git
cd pokerhole
```

If you already cloned without `--recursive`:

```bash
git submodule update --init --recursive
```

### Running the Server

```bash
cd pokerhole-server
docker compose up -d
./gradlew bootRun
```

Server will start on:
- HTTP: 8080
- TCP Terminal: 7777

### Running the Client

```bash
cd pokerhole-cli
go run cmd/pokerhole/main.go
```

Or build and install:

```bash
go build -o pokerhole cmd/pokerhole/main.go
./pokerhole
```

---

## Project Structure

```
pokerhole/
├── pokerhole-server/    # Spring Boot backend (Java 21)
│   ├── src/
│   ├── build.gradle.kts
│   └── README.md
│
└── pokerhole-cli/       # Terminal client (Go + Bubble Tea)
    ├── cmd/
    ├── internal/
    └── README.md
```

---

## Architecture

### Backend (pokerhole-server)

- Hexagonal Architecture + DDD
- Spring Boot 4.0.0-M3 with Virtual Threads
- WebSocket JSON protocol
- AI players with multiple strategies
- Automatic matchmaking

See [pokerhole-server/README.md](./pokerhole-server/README.md) for details.

### Client (pokerhole-cli)

- Modern terminal UI with Bubble Tea framework
- WebSocket client
- Cross-platform (macOS, Linux, Windows)
- User data persistence (.pokerhole/)

See [pokerhole-cli/README.md](./pokerhole-cli/README.md) for details.

---

## Development

### Server Development

```bash
cd pokerhole-server
./gradlew test
./gradlew bootRun
```

### Client Development

```bash
cd pokerhole-cli
go test ./...
go run cmd/pokerhole/main.go
```

### Updating Submodules

```bash
git submodule update --remote
```

---

## Documentation

- **Server**: [pokerhole-server/README.md](./pokerhole-server/README.md)
  - [ARCHITECTURE.md](./pokerhole-server/ARCHITECTURE.md)
  - [ROADMAP.md](./pokerhole-server/ROADMAP.md)
  - [CHANGELOG.md](./pokerhole-server/CHANGELOG.md)
  - [CONTRIBUTING.md](./pokerhole-server/CONTRIBUTING.md)

- **Client**: [pokerhole-cli/README.md](./pokerhole-cli/README.md)
  - [CLAUDE.md](./pokerhole-cli/CLAUDE.md)

---

## Tech Stack

### Server
- Spring Boot 4.0.0-M3
- Java 21 (Virtual Threads)
- PostgreSQL 16
- WebSocket
- MapStruct, Lombok

### Client
- Go 1.22+
- Bubble Tea TUI framework
- WebSocket client
- JSON protocol

---

## Contributing

1. Fork the Project
2. Create your Feature Branch (`git checkout -b feature/AmazingFeature`)
3. Commit your Changes (`git commit -m 'Add some AmazingFeature'`)
4. Push to the Branch (`git push origin feature/AmazingFeature`)
5. Open a Pull Request

See [pokerhole-server/CONTRIBUTING.md](./pokerhole-server/CONTRIBUTING.md) for detailed guidelines.

---

## License

MIT License

---

## Contact

Project Maintainer: [@xiyo](https://github.com/xiyo)

Project Link: [https://github.com/bunnyholes/pokerhole](https://github.com/bunnyholes/pokerhole)
