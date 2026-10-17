# Server config overrides

Any `*.lua` file in this folder is appended to `conf/config-local.lua` every time
the server starts, so it overrides `MMOCoreORB/bin/conf/config.lua`. Use the same
global-style keys as the generated file. For example, `server-config/zones.lua`:

```lua
-- Load fewer planets for a faster start on a small machine
ZonesEnabled = { "tutorial", "tatooine", "naboo", "corellia" }
-- Allow more characters per account
ZoneOnlineCharactersPerAccount = 3
```

Apply changes with `./swgemu restart`. `*.lua` files here are git-ignored.
