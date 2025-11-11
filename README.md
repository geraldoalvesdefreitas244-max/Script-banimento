-- sistema_de_banimento.lua
-- Script genérico de banimento com persistência JSON
-- Adapte as funções onPlayerConnect, sendMessage, kickPlayer e registro de comandos à sua plataforma.

local json = require("dkjson") -- ou outro json lib disponível; substitua se necessário

local BAN_FILE = "bans.json"

local BanSystem = {
  bans = {}, -- mapa userId -> banEntry
  admins = { "steam:111111111111111", "user:meuAdminExemplo" } -- ids dos admins (exemplo)
}

-- Helpers -------------------------------------------------------------------
local function now() return os.time() end

local function isAdmin(userId)
  for _, id in ipairs(BanSystem.admins) do
    if id == userId then return true end
  end
  return false
end

local function saveBans()
  local f, err = io.open(BAN_FILE, "w")
  if not f then
    print("Erro ao salvar bans:", err)
    return false
  end
  f:write(json.encode(BanSystem.bans, { indent = true }))
  f:close()
  return true
end

local function loadBans()
  local f = io.open(BAN_FILE, "r")
  if not f then
    BanSystem.bans = {}
    return
  end
  local content = f:read("*a")
  f:close()
  local obj, pos, err = json.decode(content)
  if err then
    print("Erro ao carregar bans.json:", err)
    BanSystem.bans = {}
  else
    BanSystem.bans = obj or {}
  end
end

-- Ban APIs ------------------------------------------------------------------
-- banEntry = { id = userId, reason = "...", bannedBy = adminId, ts = epoch, expires = epoch_or_nil }
function BanSystem.ban(userId, reason, durationSeconds, adminId)
  reason = reason or "Sem motivo informado"
  local ts = now()
  local expires = nil
  if durationSeconds and tonumber(durationSeconds) and durationSeconds > 0 then
    expires = ts + math.floor(durationSeconds)
  end
  BanSystem.bans[tostring(userId)] = {
    id = userId,
    reason = reason,
    bannedBy = adminId or "console",
    ts = ts,
    expires = expires
  }
  saveBans()
  print(("Usuário %s banido por %s (por %s segundos)"):format(userId, reason, tostring(durationSeconds)))
end

function BanSystem.unban(userId, adminId)
  if BanSystem.bans[tostring(userId)] then
    BanSystem.bans[tostring(userId)] = nil
    saveBans()
    print(("Usuário %s desbanido por %s"):format(userId, adminId or "console"))
    return true
  end
  return false
end

function BanSystem.isBanned(userId)
  local b = BanSystem.bans[tostring(userId)]
  if not b then return false, nil end
  if b.expires and now() >= b.expires then
    -- ban expirado: limpar
    BanSystem.bans[tostring(userId)] = nil
    saveBans()
    return false, nil
  end
  return true, b
end

function BanSystem.listBans()
  local out = {}
  for k,v in pairs(BanSystem.bans) do
    table.insert(out, v)
  end
  return out
end

-- Integração com servidor (adaptar conforme plataforma) ---------------------
-- Exemplos genéricos de funções que você deverá adaptar:
local function sendMessage(target, text)
  -- Adaptar: enviar mensagem ao jogador / console
  print(("Mensagem para %s: %s"):format(tostring(target), text))
end

local function kickPlayer(playerId, reason)
  -- Adaptar: kicker o jogador do servidor
  print(("Kick %s -> %s"):format(tostring(playerId), reason))
  -- Exemplo: em FiveM seria DropPlayer(playerId, reason)
end

-- Chamado quando um jogador tenta conectar; retorna true se permitido, false se negado.
function BanSystem.onPlayerConnect(userId, connectionInfo)
  local banned, entry = BanSystem.isBanned(userId)
  if banned then
    local msg = ("Você está banido.\nMotivo: %s\nBanido por: %s\nExpira: %s")
      :format(entry.reason, entry.bannedBy, entry.expires and os.date("%c", entry.expires) or "Nunca")
    -- Em servidores reais: negar conexão / mostrar mensagem
    print(("Tentativa de conexão negada a %s: %s"):format(userId, msg))
    return false, msg
  end
  return true
end

-- Comandos de administração (exemplo simples, adapte à sua engine de comandos) ---
-- Exemplo: comando /ban <userId> <tempo_segundos|perm> <motivo>
function BanSystem.handleCommand(senderId, cmd, args)
  cmd = cmd:lower()
  if cmd == "ban" then
    if not isAdmin(senderId) then return sendMessage(senderId, "Sem permissão.") end
    local target = args[1]
    local timeArg = args[2]
    local reason = table.concat(args, " ", 3) or "Sem motivo"
    local dur = nil
    if timeArg and timeArg ~= "perm" then
      dur = tonumber(timeArg)
      if not dur then return sendMessage(senderId, "Tempo inválido.") end
    end
    BanSystem.ban(target, reason, dur, senderId)
    sendMessage(senderId, ("Usuário %s banido."):format(target))
    -- se estiver online, kicker
    kickPlayer(target, ("Banido: %s"):format(reason))
    return
  elseif cmd == "unban" then
    if not isAdmin(senderId) then return sendMessage(senderId, "Sem permissão.") end
    local target = args[1]
    if not target then return sendMessage(senderId, "Use: unban <userId>") end
    local ok = BanSystem.unban(target, senderId)
    if ok then sendMessage(senderId, ("Usuário %s desbanido."):format(target))
    else sendMessage(senderId, "Usuário não está banido.") end
    return
  elseif cmd == "banlist" then
    if not isAdmin(senderId) then return sendMessage(senderId, "Sem permissão.") end
    local list = BanSystem.listBans()
    if #list == 0 then return sendMessage(senderId, "Nenhum ban ativo.") end
    for _,b in ipairs(list) do
      local exp = b.expires and os.date("%c", b.expires) or "Nunca"
      sendMessage(senderId, ("ID:%s - Motivo:%s - By:%s - Expira:%s"):format(b.id, b.reason, b.bannedBy, exp))
    end
    return
  end
end

-- Inicialização
loadBans()

-- Retornar modulo para integrar externamente
return BanSystem# Script-banimento
Moderador do Roblox
