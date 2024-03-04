---
title: Can I use scala in codespaces?
---

## Why?
Sometime it helps for reproduction or sharing, to have a "clean" environment. Also... shared environment means no local problems. We hope.

## Status:

Some stuff works. It seems that

```json
{
	"name": "Ubuntu",
	// Or use a Dockerfile or Docker Compose file. More info: https://containers.dev/guide/dockerfile
	"image": "mcr.microsoft.com/devcontainers/base:jammy",
	"customizations": {
		"vscode": {
			"extensions": [
				"scalameta.metals",
				"usernamehw.errorlens",
				"vscjava.vscode-java-pack"
			]
		}
	},

	// Features to add to the dev container. More info: https://containers.dev/features.
	"features": {
		"ghcr.io/devcontainers/features/java:1": {
			"version":21
		},
		"ghcr.io/devcontainers/features/node:1":{
		}

	},

	// Use 'forwardPorts' to make a list of ports inside the container available locally.
	// "forwardPorts": [],

	// Use 'postCreateCommand' to run commands after the container is created.
	"postCreateCommand": "yarn install & ./millw -v && ./millw __.prepareOffline && ./millw mill.bsp.BSP/install"

	// Configure tool-specific properties.
	// "customizations": {},

	// Uncomment to connect as root instead. More info: https://aka.ms/dev-containers-non-root.
	// "remoteUser": "root"
}

```
Can be a good base. It may be worth ejecting in to a custom dockerfile at some point, but not worth the time.

## Not working
the backend and the frontend start

However, the frontend cannot tunnel to the backend... so fullstack doesn't seem to work so well.

Seems otherwise pretty robust though.
