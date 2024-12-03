## react-router-entry-points

(A proposal for implementing [spatial compute](https://sunilpai.dev/posts/spatial-compute/) for [react-router](https://reactrouter.com/home) apps)

A script that generates `WorkerEntryPoint`s for every route pattern in a react-router v7 app.

Given a route definition like this:

```ts
import {
  type RouteConfig,
  index,
  layout,
  prefix,
  relative,
  route,
} from "@react-router/dev/routes";

const routes = [
  index("./home.tsx"),
  route("about", "./about.tsx"),

  layout("./auth/layout.tsx", [
    route("login", "./auth/login.tsx"),
    route("register", "./auth/register.tsx"),
  ]),

  ...prefix("concerts", [
    index("./concerts/home.tsx"),
    route(":city", "./concerts/city.tsx"),
    route("trending", "./concerts/trending.tsx"),
  ]),
];
export default routes satisfies RouteConfig;
```

We should be able to generate a wrapper around the Worker that looks like this:

```ts
import { WorkerEntryPoint } from "cloudflare:workers";
import { createRequestHandler } from "react-router";

const requestHandler = createRequestHandler(
  // @ts-expect-error - virtual module provided by React Router at build time
  () => import("virtual:react-router/server-build"),
  import.meta.env.MODE
);

const URLPatterns = {
  "/": 'IndexHome',
  "/about": 'About',
  "/login": 'AuthLogin',
  "/register": 'AuthRegister',
  "/concerts": 'ConcertsHome',
  "/concerts/:city": 'ConcertsCity',
  "/concerts/trending": 'ConcertsTrending',
}

export const IndexHome extends WorkerEntryPoint {
  fetch(request){
    return requestHandler(request, this.env, this.ctx);
  }
}

export const About extends WorkerEntryPoint {
  fetch(request){
    return requestHandler(request, this.env, this.ctx);
  }
}

// ... and so on

export default {
  async fetch(request, env, ctx){
    const url = new URL(request.url);
    for(const [pattern, EntryPoint] of Object.entries(URLPatterns)){
      if(url.pathname.match(pattern)){
        return env.SELF.[EntryPoint].fetch(request, env, ctx);
      }
    }
  }
} satisfies ExportedHandler<Env>

```

## Why???

Cloudflare is implementing 2 features that will make this useful:

- Self referential bindings: Instead of needing to declare every entypoint in `wrangler.toml`, we'll be able to reference them by name (hence `env.SELF`)

- Per-entrypoint "smart placement" of workers: This will automatically locate a `WorkerEntryPoint` close to the data sources that it depends on.

This script will generate a `WorkerEntryPoint` for each route pattern, and then wire up the `fetch` handler to route to the correct entrypoint based on the URL pathname. So portions of the app will physically locate themselves next to the data sources that they depend on, optimising without any manual configuration.
