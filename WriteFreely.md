---
name: Library/zefhemel/WriteFreely
tags: meta/library
---
Implements basic [[^Library/Std/Infrastructure/Share]] for [WriteFreely](https://writefreely.org) ([API docs](https://developers.write.as/docs/api/)).

# Config example
Configure, e.g. in your [[CONFIG]]:
```lua
config.set("writefreely", {
  blog = {
    url = "https://write.as", -- or whever your blog is published
    username = "youruser",
    password = "yourpassword",
    collection = "youruser" -- usually this is your user name
  }
})
```

# API implementation
```space-lua
writefreely = {}
-- returns a config object
function writefreely.auth(url, username, password, collection)
  local tokenResp = net.proxyFetch(url .. "/api/auth/login", {
    method = "POST",
    headers = {
      ["Content-type"] = "application/json"
    },
    body = {
      alias = username,
      pass = password,
      collection = collection
    }
  })
  if tokenResp.ok then
    return {
      url = url,
      token = tokenResp.body.data.access_token,
      collection = collection
    }
  else
    print("Error", tokenResp)
    error("Failed")
  end
end

local function request(cfg, api, method, body)
  local resp = net.proxyFetch(cfg.url .. api, {
    method = method,
    headers = {
      Authorization = "Token " .. cfg.token,
      ["Content-Type"] = "application/json"
    },
    body = body
  })
  return resp.body
end


-- post keys:
--   body
--   title
--   slug
--   font
function writefreely.publish(cfg, post)
  local req = request(cfg, "/api/collections/" .. cfg.collection .. "/posts", "POST", post)
  if req.code != 201 then
    error(req.error_msg)
  end
  return req.data
end

function writefreely.update(cfg, id, post)
  local req = request(cfg, "/api/posts/" .. id, "POST", post)
  if req.code != 200 then
    error(req.error_msg)
  end
  return req.data
end

function writefreely.get(cfg, slug)
  local req = request(cfg, "/api/collections/" .. cfg.collection .. "/posts/" .. slug, "GET")
  if req.code == 200 then
    return req.data
  else
    return nil
  end
end

function writefreely.createOrUpdate(cfg, post)
  local existing = writefreely.get(cfg, post.slug)
  if not existing then
    print("New post")
    writefreely.publish(cfg, post)
  else
    print("Updating post")
    writefreely.update(cfg, existing.id, {
      title = post.title,
      body = post.body
    })
  end
end

service.define {
  selector = "share:onboard",
  match = {
    name = "WriteFreely post",
    descripion = "Publish this page as a WriteFreely post"
  },
  run = function(data)
    local wfConfig = config.get("writefreely")
    if not wfConfig then
      editor.flashNotification("No WriteFreely blogs configured", "error")
      return
    end
    local availableBlogs = table.keys(wfConfig)
    if #availableBlogs == 0 then
      editor.flashNotification("No blogs configured", "error")
      return
    end
    -- Prompt (if required) for the blog to publish to
    local cfgName, cfg
    if #availableBlogs > 1 then
      local options = {}
      for cfgName, cfg in pairs(wfConfig) do
        table.insert(options, {
          name = cfgName,
          cfg = cfg
        })
      end
      local cfg = editor.filterBox("Blog to publish to", options, "Select the blog of your desire")
      if not cfg then
        return
      end
      cfgName = cfg.name
      cfg = cfg.cfg
    else
      cfgName = availableBlogs[1]
      cfg = wfConfig[cfgName]
    end
    -- Now ask for a slug
    local slug = editor.prompt("Post slug:", "my-new-post")
    if not slug then
      return
    end
    -- Write an initial version using writeURI
    local uri = "writefreely:" .. cfgName .. ":" .. slug
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
--   writefreely:blog-name:slug
service.define {
  selector = "net.writeURI:writefreely:*",
  match = {priority = 10},
  run = function(data)
    local uri = data.uri
    local text = data.content
    local _, cfgName, slug = table.unpack(uri:split(":"))
    if not cfgName or not slug then
      error("Invalid writefreely URI: expected format is writefreely:blog-name:slug")
    end
    local cfg = config.get("writefreely")[cfgName]
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
    local wf = writefreely.auth(cfg.url, cfg.username, cfg.password, cfg.collection)
    writefreely.createOrUpdate(wf, {
      slug = slug,
      title = title,
      body = text,
    })
  end
}

```
