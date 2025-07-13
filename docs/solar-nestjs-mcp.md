# NestJS Solar MCP Server Example

This guide outlines one approach to integrate the SDK with a NestJS application that exposes an MCP server capable of returning solar estimates and quotes.

## Architecture Overview

```
Client (Streamable HTTP)
        │
        ▼
[MCP Controller] ──▶ [McpServer] ──▶ [SolarService]
        │                 │
        │                 └── Calls Solar API & business logic
        ▼
  Response via Streamable HTTP
```

- **MCP Controller** – Defines the `/mcp` endpoints for POST/GET/DELETE. Each request is passed to `StreamableHTTPServerTransport` so responses can be streamed back to the client.
- **McpServer** – Instance from the SDK. Tools/prompts are registered here.
- **SolarService** – Encapsulates calls to third‑party solar APIs and your own quote logic.

## Setup Steps

1. Install the SDK as a dependency in your NestJS project:
   ```bash
   npm install @modelcontextprotocol/sdk
   ```
2. Create a module that provides the `McpServer` and registers your solar tools.
3. Implement a controller that delegates HTTP requests to a `StreamableHTTPServerTransport` instance.
4. Use a NestJS service to integrate with any external solar APIs and implement the quoting logic.

## Example Implementation

### mcp.module.ts
```typescript
import { Module, OnModuleInit } from '@nestjs/common';
import { McpServer } from '@modelcontextprotocol/sdk/server/mcp.js';
import { StreamableHTTPServerTransport } from '@modelcontextprotocol/sdk/server/streamableHttp.js';
import { SolarService } from './solar.service.js';

@Module({
  providers: [SolarService],
  controllers: [],
})
export class McpModule implements OnModuleInit {
  private server = new McpServer({ name: 'solar-mcp', version: '1.0.0' });

  onModuleInit() {
    this.server.registerTool(
      'solar-estimate',
      {
        title: 'Get Solar Estimate',
        description: 'Returns production and cost estimates',
        inputSchema: { address: z.string() },
      },
      async ({ address }) => ({
        content: await this.solarService.getEstimate(address),
      }),
    );

    this.server.registerTool(
      'solar-quote',
      {
        title: 'Get Solar Quote',
        description: 'Returns a full quote based on system size and rebates',
        inputSchema: { address: z.string(), systemSizeKw: z.number() },
      },
      async (args) => ({
        content: await this.solarService.getQuote(args),
      }),
    );
  }

  connectTransport(transport: StreamableHTTPServerTransport) {
    this.server.connect(transport);
  }
}
```

### solar.service.ts
```typescript
import { Injectable } from '@nestjs/common';
import fetch from 'node-fetch';

@Injectable()
export class SolarService {
  async getEstimate(address: string) {
    // Call your solar API here
    const response = await fetch(`https://api.example.com/estimate?address=${encodeURIComponent(address)}`);
    return response.json();
  }

  async getQuote(input: { address: string; systemSizeKw: number }) {
    // Business logic and additional API calls
    const response = await fetch('https://api.example.com/quote', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify(input),
    });
    return response.json();
  }
}
```

### mcp.controller.ts
```typescript
import { Controller, Post, Get, Delete, Req, Res } from '@nestjs/common';
import { Request, Response } from 'express';
import { StreamableHTTPServerTransport } from '@modelcontextprotocol/sdk/server/streamableHttp.js';
import { McpModule } from './mcp.module.js';

@Controller('mcp')
export class McpController {
  constructor(private readonly mcpModule: McpModule) {}

  @Post()
  async handlePost(@Req() req: Request, @Res() res: Response) {
    const transport = new StreamableHTTPServerTransport({
      sessionIdGenerator: () => req.headers['mcp-session-id'] as string ?? undefined,
    });
    this.mcpModule.connectTransport(transport);
    await transport.handleRequest(req, res, req.body);
  }

  @Get()
  async handleGet(@Req() req: Request, @Res() res: Response) {
    const transport = new StreamableHTTPServerTransport({});
    this.mcpModule.connectTransport(transport);
    await transport.handleRequest(req, res);
  }

  @Delete()
  async handleDelete(@Req() req: Request, @Res() res: Response) {
    const transport = new StreamableHTTPServerTransport({});
    this.mcpModule.connectTransport(transport);
    await transport.handleRequest(req, res);
  }
}
```

### main.ts
```typescript
import { NestFactory } from '@nestjs/core';
import { McpModule } from './mcp.module.js';
import { McpController } from './mcp.controller.js';

async function bootstrap() {
  const app = await NestFactory.createApplicationContext(McpModule);
  const controller = app.get(McpController);
  app.use('/mcp', controller);
}

bootstrap();
```

This example shows the minimal wiring required. In production you would store transports by session ID (like the SDK examples do) and enable DNS rebinding protection.

## Tooling Recommendations

- **NestJS** for the application framework
- **node-fetch** or Axios for external HTTP calls
- **Zod** for input validation (already a dependency of the SDK)
- **StreamableHTTPServerTransport** for communication between client and server
- **McpServer** for managing tools and prompts

## Next Steps

1. Expand `SolarService` with your full business logic and caching.
2. Persist session information if you need resumability across multiple nodes.
3. Expose any additional tools or prompts (e.g., `getRebates`, `scheduleInstallation`).
4. Secure the endpoints with OAuth middleware as shown in other examples.

