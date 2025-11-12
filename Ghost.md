---
name: Library/zefhemel/Ghost
tags: meta/library
---
Implements basic [[^Library/Std/Infrastructure/Share]] for [Ghost](https://ghost.org/). Currently only posts are supported.

# Config example
You need to create a Custom Integration key for this. Go to your Ghost admin panel > Advanced > Ingegrations > Custom > Add custom integration. Generate one, then copy the Admin API key and add a config somewhere in your SilverBullet space (e.g. in your [[CONFIG]] page) with `space-lua`:
```lua
config.set("ghost", {
  blog = {
    url = "https://xxx.ghost.io",
    key = "xxx"
  }
})
```

# Publishing a post
You can now publish any page as a Ghost post with the `Share: Page` command. You will be guided through the steps when you select the “Ghost post” option.

# Implementation
Sadly, the JavaScript Ghost Admin library doesn’t seem to work in the browser, therefore we have to fall back to custom implemented API calls.

```space-lua
ghost = {}

-- Convert hex string to byte array
local function fromHex(hexString)
  local bytes = {}
  
  for i = 1, #hexString, 2 do
      local byteValue = tonumber(hexString:sub(i, i + 1), 16)
      table.insert(bytes, byteValue)
  end
  return js.new(js.window.Uint8Array, bytes)
end

-- Generate JWT with the jose library
local function genToken(key)
  local jose = js.import "https://esm.sh/jose@6.0.10"

  local pieces = string.split(key, ":")

  local jwt = js.new(jose.SignJWT, {
    ['urn:example:claim'] = true
  }).setProtectedHeader({ alg = 'HS256', kid = pieces[1] })
    .setIssuedAt()
    .setAudience('/admin/')
    .setExpirationTime('5m')
    .sign(fromHex(pieces[2]))

  return jwt
end

-- Authenticated admin Ghost request
local function authenticatedRequest(cfg, path, method, body)
  local token = genToken(cfg.key)
  return net.proxyFetch(cfg.url .. path, {
    method = method,
    headers = {
      Authorization = "Ghost " .. token,
      ["Content-Type"] = "application/json",
      Origin = cfg.url,
      ["Accept-Version"] = "v5.0"
    },
    body = body
  })
end

-- Create lexical object for markdown
local function markdownToLexical(text)
  return js.window.JSON.stringify {
    root = {
      children = {
        {
          type = "markdown",
          version = 1,
          markdown = text,
        }
      },
      direction = nil,
      format = "",
      indent = 0,
      type = "root",
      version = 1,
    }
  }
end

service.define {
  selector = "share:onboard",
  match = {
    name = "Ghost post",
    descripion = "Publish this page as a post on the Ghost CMS"
  },
  run = function(data)
    local ghostConfig = config.get("ghost")
    if not ghostConfig then
      editor.flashNotification("No Ghost blogs configured", "error")
      return
    end
    local availableBlogs = table.keys(ghostConfig)
    if #availableBlogs == 0 then
      editor.flashNotification("No Ghost blogs configured", "error")
      return
    end
    -- Prompt (if required) for the blog to publish to
    local cfgName, cfg
    if #availableBlogs > 1 then
      local options = {}
      for cfgName, cfg in pairs(ghostConfig) do
        table.insert(options, {
          name = cfgName,
          cfg = cfg
        })
      end
      local cfg = editor.filterBox("Ghost blog to publish to", options, "Select the Ghost blog of your desire")
      if not cfg then
        return
      end
      cfgName = cfg.name
      cfg = cfg.cfg
    else
      cfgName = availableBlogs[1]
      cfg = ghostConfig[cfgName]
    end
    -- Now ask for a sluge
    local slug = editor.prompt("Post slug:", "my-new-post")
    if not slug then
      return
    end
    -- Write an initial version using writeURI
    local uri = "ghost:" .. cfgName .. ":" .. slug
    local text = editor.getText()
    net.writeURI(uri, text)
    return {
      uri = uri,
      hash = share.contentHash(text),
      mode = "push"
    }
  end
}

-- URIs scheme support
--   ghost:blog-name:slug
service.define {
  selector = "net.writeURI:ghost:*",
  match = {priority = 10},
  run = function(data)
    local uri = data.uri
    local text = data.content
    local _, cfgName, slug = table.unpack(uri:split(":"))
    if not cfgName or not slug then
      error("Invalid ghost URI: expected format is ghost:blog-name:slug")
    end
    local cfg = config.get("ghost")[cfgName]
    if not cfg then
      error("Could not find config: " .. cfgName)
    end
    -- Note: we need the page name, which cannot be extracted from this API call so need to assume it's triggered right from the page
    local pageName = editor.getCurrentPage()
    local pageNameBits = pageName:split("/")
    -- Last bit is assumed to be the title
    local title = pageNameBits[#pageNameBits]
    local fm = index.extractFrontmatter(text, {
      removeFrontMatterSection=true
    })
    local frontmatter = fm.frontmatter
    text = fm.text
    title = title or frontmatter.title
    -- Let's check if this posts was already created
    local r = authenticatedRequest(cfg, "/ghost/api/admin/posts/slug/" .. slug .. "/", "GET")
    if r.status == 404 then
      -- Doesn't exist yet, let's create
      local r = authenticatedRequest(cfg, "/ghost/api/admin/posts/", "POST", {
        posts = {{
          title = title,
          slug = slug,
          lexical = markdownToLexical(text),
          status = "draft"
        }}
      })
      if r.ok then
        editor.flashNotification "Created post successfully"
      else
        editor.flashNotification("Failed to create post, check logs", "error")
        js.log("Error", r)
      end
    else
      -- Existing post, let's update
      local oldPost = r.body.posts[1]
      local updatedAt = oldPost.updated_at
      local r = authenticatedRequest(cfg, "/ghost/api/admin/posts/" .. oldPost.id, "PUT", {
        posts = {{
          title = title,
          slug = slug,
          lexical = markdownToLexical(text),
          updated_at = updatedAt
        }}
      })
      if not r.ok then
        editor.flashNotification("Failed to update post, check logs", "error")
        js.log("Error", r)
      end
    end
  end
}

```
