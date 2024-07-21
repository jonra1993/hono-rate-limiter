# <div align="center">🔥`@hono-rate-limiter/cloudflare`🔥</div>

<div align="center">

[![tests](https://img.shields.io/github/actions/workflow/status/rhinobase/hono-rate-limiter/test.yml)](https://github.com/rhinobase/hono-rate-limiter/actions/workflows/test.yml)
[![npm version](https://img.shields.io/npm/v/@hono-rate-limiter/cloudflare.svg)](https://npmjs.org/package/@hono-rate-limiter/cloudflare "View this project on NPM")
[![npm downloads](https://img.shields.io/npm/dm/@hono-rate-limiter/cloudflare)](https://www.npmjs.com/package/@hono-rate-limiter/cloudflare)
[![license](https://img.shields.io/npm/l/%40hono-rate-limiter%2Fcloudflare)](LICENSE)

</div>

This package include [`WorkersKV`](https://developers.cloudflare.com/kv/) store for the [`hono-rate-limiter`](https://github.com/rhinobase/hono-rate-limiter) middleware and `cloudflareRateLimiter` func for [Workers Rate Limiting](https://developers.cloudflare.com/workers/runtime-apis/bindings/rate-limit/) API.

> [!NOTE]  
> The `keyGenerator` function and `rateLimitBinding` bindings needs to be defined for `cloudflareRateLimiter` to work properly in your environment. Please ensure that you define the `keyGenerator` function according to the documentation before using the library.

## Installation

```sh
# Using npm/yarn/pnpm/bun
npm add @hono-rate-limiter/cloudflare
```

## Usage

### Examples

#### Using `cloudflareRateLimiter`

```toml
# wrangler.toml
# The rate limiting API is in open beta.
[[unsafe.bindings]]
name = "MY_RATE_LIMITER"
type = "ratelimit"
# An identifier you define, that is unique to your Cloudflare account.
# Must be an integer.
namespace_id = "1001"

# Limit: the number of tokens allowed within a given period in a single
# Cloudflare location
# Period: the duration of the period, in seconds. Must be either 10 or 60
simple = { limit = 100, period = 60 }
```

For more info on setting up your Workers Rate Limiting API you can check out [Rate Limiting Guide](https://developers.cloudflare.com/workers/runtime-apis/bindings/rate-limit/) on cloudflare.

```ts
import { cloudflareRateLimiter, RateLimitBinding } from "@hono-rate-limiter/cloudflare";

type AppType = {
  Variables: {
    rateLimit: boolean;
  };
  Bindings: {
    MY_RATE_LIMITER: RateLimitBinding;
  };
};

// Apply the rate limiting middleware to all requests.
const app = new Hono<AppType>().use((c: Context, next: Next) =>
  cloudflareRateLimiter<AppType>({
    rateLimitBinding: c.env.MY_RATE_LIMITER,
    keyGenerator: (c) => c.req.header("cf-connecting-ip") ?? "", // Method to generate custom identifiers for clients.
  })(c, next)
);
```

#### Using `WorkersKVStore`

```toml
# wrangler.toml
[[kv_namespaces]]
binding = "CACHE"
id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

For more info on setting up your `WorkersKV` you can check out [Get Started Guide](https://developers.cloudflare.com/kv/get-started) on cloudflare.

```ts
// index.ts
import { WorkersKVStore } from "@hono-rate-limiter/cloudflare";
import { rateLimiter } from "hono-rate-limiter";
import { Context, Next } from "hono";
import { KVNamespace } from "cloudflare:worker";

// Add this in Hono app
type Bindings = {
  CACHE: KVNamespace;
  // ... other binding types
};

const app = new Hono<{ Bindings: Bindings }>();

// Apply the rate limiting middleware to all requests.
app.use((c: Context, next: Next) =>
  rateLimiter<{ Bindings: Bindings }>({
    windowMs: 15 * 60 * 1000, // 15 minutes
    limit: 100, // Limit each IP to 100 requests per `window` (here, per 15 minutes).
    standardHeaders: "draft-6", // draft-6: `RateLimit-*` headers; draft-7: combined `RateLimit` header
    keyGenerator: (c) => c.req.header("cf-connecting-ip") ?? "", // Method to generate custom identifiers for clients.
    store: new WorkersKVStore({ namespace: c.env.CACHE }), // Here CACHE is your WorkersKV Binding.
  })(c, next)
);
```

### Configuration

#### `namespace`

The KV namespace to use. The value you set for <BINDING_NAME> will be used to reference this database in your Worker.

#### `prefix`

The text to prepend to the key in Redis.

Defaults to `hrl:`.

## Contributing

We would love to have more contributors involved!

To get started, please read our [Contributing Guide](https://github.com/rhinobase/hono-rate-limiter/blob/main/CONTRIBUTING.md).
