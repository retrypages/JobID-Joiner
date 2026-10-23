-- jobid joiner on github

--[[
	JOB JOINER PRO - mobile first (Delta / Android)
	- 1 requisicao por ID (debounce + cache), nao 1 por tecla
	- paginacao da lista de servidores (300 servidores) em vez de 100
	- clipboard real, validacao de UUID, feedback por linha de status
]]

local Players  = game:GetService("Players")
local Tween    = game:GetService("TweenService")
local Http     = game:GetService("HttpService")
local Teleport = game:GetService("TeleportService")
local UIS      = game:GetService("UserInputService")

local lp      = Players.LocalPlayer
local isTouch = UIS.TouchEnabled and not UIS.KeyboardEnabled

local C = {
	bg      = Color3.fromRGB(11, 12, 15),
	head    = Color3.fromRGB(16, 18, 22),
	field   = Color3.fromRGB(20, 22, 28),
	surf    = Color3.fromRGB(26, 29, 36),
	line    = Color3.fromRGB(37, 41, 50),
	text    = Color3.fromRGB(232, 235, 242),
	dim     = Color3.fromRGB(124, 132, 148),
	accent  = Color3.fromRGB(86, 156, 255),
	accent2 = Color3.fromRGB(0, 201, 255),
	ok      = Color3.fromRGB(63, 205, 145),
	bad     = Color3.fromRGB(240, 100, 100),
}

local function mk(class, props, parent)
	local o = Instance.new(class)
	for k, v in next, props do o[k] = v end
	o.Parent = parent
	return o
end

local function tween(o, props, t, style, dir)
	Tween:Create(o, TweenInfo.new(t or 0.12, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out), props):Play()
end

local function corner(o, r)
	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(0, r or 10)
	c.Parent = o
	return c
end

local function tapFx(b, to)
	local sc = Instance.new("UIScale")
	sc.Parent = b
	b.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			tween(sc, { Scale = to or 0.94 }, 0.07)
		end
	end)
	b.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			tween(sc, { Scale = 1 }, 0.12, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
		end
	end)
end

local conns = {}

local function on(sig, fn)
	local c = sig:Connect(fn)
	conns[#conns + 1] = c
	return c
end

local function stroke(o, color, alpha)
	local s = Instance.new("UIStroke")
	s.Color, s.Transparency, s.Thickness = color, alpha or 0, 1
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	s.Parent = o
	return s
end

local function getHost()
	local ok, h = pcall(function() return gethui and gethui() end)
	if ok and h then return h end
	ok, h = pcall(function() return game:GetService("CoreGui") end)
	if ok and h then return h end
	return lp:WaitForChild("PlayerGui")
end

local host = getHost()
for _, obj in ipairs(host:GetChildren()) do
	if obj.Name == "JobJoinerPro" and obj:IsA("ScreenGui") then obj:Destroy() end
end

-- layout ---------------------------------------------------------------
local cam = workspace.CurrentCamera
local vx  = cam and cam.ViewportSize.X or 400
local W   = math.clamp(vx - 40, 272, 340)
local H   = 268
local BODY_H = 224
local bx  = 12
local bw  = W - 24

local sg = mk("ScreenGui", {
	Name = "JobJoinerPro",
	ResetOnSpawn = false,
	IgnoreGuiInset = true,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	DisplayOrder = 2147483647,
}, nil)
if not pcall(function() sg.Parent = host end) then
	sg.Parent = lp:WaitForChild("PlayerGui")
end

local mf = mk("Frame", {
	Name = "Root",
	AnchorPoint = Vector2.new(0.5, 0.5),
	Position = UDim2.fromScale(0.5, 0.45),
	Size = UDim2.fromOffset(W, H),
	BackgroundColor3 = C.bg,
	BorderSizePixel = 0,
	ClipsDescendants = true,
	Active = true,
}, sg)
corner(mf, 12)
stroke(mf, C.line, 0.15)

local head = mk("Frame", {
	Size = UDim2.new(1, 0, 0, 44),
	BackgroundColor3 = C.head,
	BorderSizePixel = 0,
}, mf)

mk("Frame", {
	Size = UDim2.new(1, 0, 0, 1),
	Position = UDim2.new(0, 0, 1, -1),
	BackgroundColor3 = C.line,
	BorderSizePixel = 0,
}, head)

local logo = mk("Frame", {
	Size = UDim2.fromOffset(6, 6),
	Position = UDim2.new(0, 14, 0.5, -3),
	BackgroundColor3 = C.accent,
	BorderSizePixel = 0,
}, head)
corner(logo, 999)

mk("TextLabel", {
	Size = UDim2.new(1, -110, 1, 0),
	Position = UDim2.fromOffset(26, 0),
	BackgroundTransparency = 1,
	Text = "Job Joiner",
	TextColor3 = C.text,
	TextSize = 13,
	Font = Enum.Font.GothamBold,
	TextXAlignment = Enum.TextXAlignment.Left,
}, head)

local minBtn = mk("TextButton", {
	Size = UDim2.fromOffset(26, 26),
	Position = UDim2.new(1, -66, 0.5, -13),
	BackgroundColor3 = C.surf,
	AutoButtonColor = false,
	Text = "-",
	TextColor3 = C.dim,
	TextSize = 14,
	TextYAlignment = Enum.TextYAlignment.Center,
	Font = Enum.Font.GothamBold,
}, head)
corner(minBtn, 7)
tapFx(minBtn)

local closeBtn = mk("TextButton", {
	Size = UDim2.fromOffset(26, 26),
	Position = UDim2.new(1, -36, 0.5, -13),
	BackgroundColor3 = C.surf,
	AutoButtonColor = false,
	Text = "X",
	TextColor3 = C.dim,
	TextSize = 11,
	Font = Enum.Font.GothamBold,
}, head)
corner(closeBtn, 7)
tapFx(closeBtn)

local body = mk("Frame", {
	Size = UDim2.new(1, 0, 0, BODY_H),
	Position = UDim2.fromOffset(0, 44),
	BackgroundTransparency = 1,
}, mf)

local scale = Instance.new("UIScale")
scale.Scale = 0.94
scale.Parent = mf

-- input ----------------------------------------------------------------
mk("TextLabel", {
	Size = UDim2.fromOffset(bw, 12),
	Position = UDim2.fromOffset(bx, 12),
	BackgroundTransparency = 1,
	Text = "JOB ID",
	TextColor3 = C.dim,
	TextSize = 10,
	Font = Enum.Font.GothamBold,
	TextXAlignment = Enum.TextXAlignment.Left,
}, body)

local field = mk("Frame", {
	Size = UDim2.fromOffset(bw, 42),
	Position = UDim2.fromOffset(bx, 32),
	BackgroundColor3 = C.field,
	BorderSizePixel = 0,
}, body)
corner(field, 10)
local fstroke = stroke(field, C.line, 0.15)

local box = mk("TextBox", {
	Size = UDim2.new(1, -68, 1, 0),
	Position = UDim2.fromOffset(12, 0),
	BackgroundTransparency = 1,
	Text = "",
	PlaceholderText = "Cole o Job ID",
	PlaceholderColor3 = Color3.fromRGB(96, 104, 120),
	TextColor3 = C.text,
	TextSize = 12,
	Font = Enum.Font.Code,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextTruncate = Enum.TextTruncate.AtEnd,
	ClearTextOnFocus = false,
}, field)

local pasteBtn = mk("TextButton", {
	Size = UDim2.fromOffset(48, 28),
	AnchorPoint = Vector2.new(1, 0.5),
	Position = UDim2.new(1, -7, 0.5, 0),
	BackgroundColor3 = C.surf,
	AutoButtonColor = false,
	Text = "COLAR",
	TextColor3 = C.dim,
	TextSize = 9,
	Font = Enum.Font.GothamBold,
}, field)
corner(pasteBtn, 7)
tapFx(pasteBtn, 0.92)

local srow = mk("Frame", {
	Size = UDim2.fromOffset(bw, 18),
	Position = UDim2.fromOffset(bx, 82),
	BackgroundTransparency = 1,
}, body)

local dot = mk("Frame", {
	Size = UDim2.fromOffset(6, 6),
	Position = UDim2.fromOffset(0, 6),
	BackgroundColor3 = C.dim,
	BorderSizePixel = 0,
}, srow)
corner(dot, 999)

local stMsg = mk("TextLabel", {
	Size = UDim2.new(1, -74, 1, 0),
	Position = UDim2.fromOffset(12, 0),
	BackgroundTransparency = 1,
	Text = "Servidor atual",
	TextColor3 = C.dim,
	TextSize = 10,
	Font = Enum.Font.Gotham,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextTruncate = Enum.TextTruncate.AtEnd,
}, srow)

local stNum = mk("TextLabel", {
	Size = UDim2.fromOffset(62, 18),
	AnchorPoint = Vector2.new(1, 0),
	Position = UDim2.new(1, 0, 0, 0),
	BackgroundTransparency = 1,
	Text = "0/0",
	TextColor3 = C.dim,
	TextSize = 10,
	Font = Enum.Font.Code,
	TextXAlignment = Enum.TextXAlignment.Right,
}, srow)

-- buttons --------------------------------------------------------------
local function button(label, w, h, x, y, primary)
	local b = mk("TextButton", {
		Size = UDim2.fromOffset(w, h),
		Position = UDim2.fromOffset(x, y),
		BackgroundColor3 = primary and C.accent or C.surf,
		AutoButtonColor = false,
		Text = label,
		TextColor3 = primary and Color3.new(1, 1, 1) or C.text,
		TextSize = primary and 12 or 11,
		Font = Enum.Font.GothamBold,
		TextTruncate = Enum.TextTruncate.AtEnd,
	}, body)
	corner(b, primary and 10 or 9)
	if not primary then stroke(b, C.line, 0.2) end

	local sc = Instance.new("UIScale")
	sc.Parent = b

	local base = b.BackgroundColor3
	if not isTouch then
		b.MouseEnter:Connect(function() tween(b, { BackgroundColor3 = primary and Color3.fromRGB(104, 172, 255) or Color3.fromRGB(33, 37, 46) }, 0.1) end)
		b.MouseLeave:Connect(function() tween(b, { BackgroundColor3 = base }, 0.1) end)
	end
	b.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			tween(sc, { Scale = 0.96 }, 0.07)
		end
	end)
	b.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			tween(sc, { Scale = 1 }, 0.12, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
		end
	end)
	return b
end

local joinBtn = button("Entrar no servidor", bw, 42, bx, 108, true)
local grad = Instance.new("UIGradient")
grad.Color = ColorSequence.new(C.accent, C.accent2)
grad.Parent = joinBtn

local half = (bw - 8) / 2
local copyBtn = button("Copiar meu ID", half, 38, bx, 158)
local fullBtn = button("Servidor cheio", half, 38, bx + half + 8, 158)

mk("TextLabel", {
	Size = UDim2.fromOffset(bw, 12),
	Position = UDim2.fromOffset(bx, 204),
	BackgroundTransparency = 1,
	Text = isTouch and "Arraste pelo topo - segure o campo para colar" or "RightShift recolhe - Enter entra",
	TextColor3 = Color3.fromRGB(88, 96, 112),
	TextSize = 10,
	Font = Enum.Font.Gotham,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextTruncate = Enum.TextTruncate.AtEnd,
}, body)

-- helpers --------------------------------------------------------------
local function strip(s)
	return (tostring(s):gsub("%s+", ""))
end

local function validId(id)
	return #id == 36 and id:match("^%x%x%x%x%x%x%x%x%-%x%x%x%x%-%x%x%x%x%-%x%x%x%x%-%x%x%x%x%x%x%x%x%x%x%x%x$") ~= nil
end

local function httpGet(url)
	local ok, body = pcall(function() return game:HttpGet(url) end)
	if ok and type(body) == "string" and #body > 0 then return body end
	local ok2, res = pcall(request, { Url = url, Method = "GET" })
	if ok2 and type(res) == "table" and type(res.Body) == "string" then return res.Body end
end

local cache, CACHE_TTL = {}, 12

local function servers(placeId)
	local hit = cache[placeId]
	if hit and os.clock() - hit.at < CACHE_TTL then return hit.list end

	local list, cursor = {}, nil
	for _ = 1, 3 do
		local url = string.format("https://games.roblox.com/v1/games/%d/servers/Public?limit=100", placeId)
		if cursor then url = url .. "&cursor=" .. Http:UrlEncode(cursor) end

		local raw = httpGet(url)
		if not raw then break end

		local ok, data = pcall(function() return Http:JSONDecode(raw) end)
		if not ok or type(data) ~= "table" then break end

		for _, s in ipairs(data.data or {}) do list[#list + 1] = s end
		cursor = data.nextPageCursor
		if not cursor then break end
		task.wait(0.08)
	end

	if #list > 0 then cache[placeId] = { at = os.clock(), list = list } end
	return list
end

local function findServer(id)
	for _, s in ipairs(servers(game.PlaceId)) do
		if s.id == id then return s end
	end
end

-- state ----------------------------------------------------------------
local token = 0
local closed = false

local function setState(msg, color)
	dot.BackgroundColor3 = color
	stMsg.Text = msg
	stMsg.TextColor3 = color
end

local function setCount(a, b)
	stNum.Text = (a and b) and (a .. "/" .. b) or "—"
	stNum.TextColor3 = (a and b) and C.text or C.dim
end

local function localPlayers()
	return #Players:GetPlayers(), Players.MaxPlayers
end

local function inspect(raw)
	if closed then return end
	local id = strip(raw)
	token = token + 1
	local mine = token

	if id == "" or id == game.JobId then
		local p, m = localPlayers()
		setState(id == "" and "Servidor atual" or "Você está aqui", id == "" and C.dim or C.ok)
		setCount(p, m)
		return
	end
	if #id ~= 36 then
		setState("Job ID incompleto", C.dim)
		setCount(nil)
		return
	end

	setState("Procurando servidor", C.dim)
	setCount(nil)

	task.spawn(function()
		local srv = findServer(id)
		if closed or mine ~= token then return end
		if srv then
			setState("Servidor encontrado", C.ok)
			setCount(srv.playing, srv.maxPlayers)
		else
			setState("Privado ou não listado", C.bad)
			setCount(nil)
		end
	end)
end

local function flash(msg, color)
	if closed then return end
	token = token + 1
	local mine = token
	setState(msg, color)
	task.delay(2, function()
		if mine == token and not closed then inspect(box.Text) end
	end)
end

local busy = false
local function resetJoin()
	busy = false
	joinBtn.Text = "Entrar no servidor"
end

local function join(raw)
	local id = strip(raw)
	if busy or closed then return end
	if id == "" then return flash("Cole um Job ID", C.bad) end
	if not validId(id) then return flash("Job ID inválido", C.bad) end
	if id == game.JobId then return flash("Você já está aqui", C.dim) end

	busy = true
	joinBtn.Text = "Conectando..."
	setState("Teleportando...", C.accent)

	local ok = pcall(function()
		Teleport:TeleportToPlaceInstance(game.PlaceId, id, lp)
	end)
	if not ok then
		resetJoin()
		flash("Falha ao teleportar", C.bad)
		return
	end
	task.delay(8, function() if busy then resetJoin() end end)
end

local function setClip(text)
	local ok = pcall(function() setclipboard(text) end)
	if ok then return true end
	ok = pcall(function() toclipboard(text) end)
	if ok then return true end
	ok = pcall(function() writeclipboard(text) end)
	return ok
end

local function readClip()
	local ok, v = pcall(function() return getclipboard() end)
	if ok and type(v) == "string" then return v end
	ok, v = pcall(function() return readclipboard() end)
	if ok and type(v) == "string" then return v end
end

-- interactions ---------------------------------------------------------
box:GetPropertyChangedSignal("Text"):Connect(function()
	token = token + 1
	local mine = token
	task.delay(0.35, function()
		if mine == token then inspect(box.Text) end
	end)
end)

box.Focused:Connect(function()
	tween(fstroke, { Color = C.accent, Transparency = 0 }, 0.12)
end)

box.FocusLost:Connect(function(enter)
	tween(fstroke, { Color = C.line, Transparency = 0.15 }, 0.12)
	inspect(box.Text)
	if enter then join(box.Text) end
end)

joinBtn.MouseButton1Click:Connect(function() join(box.Text) end)

copyBtn.MouseButton1Click:Connect(function()
	if setClip(game.JobId) then
		flash("ID copiado", C.ok)
	else
		box.Text = game.JobId
		flash("ID colocado no campo", C.dim)
	end
end)

pasteBtn.MouseButton1Click:Connect(function()
	local v = readClip()
	if v then
		local found = tostring(v):match("%x%x%x%x%x%x%x%x%-%x%x%x%x%-%x%x%x%x%-%x%x%x%x%-%x%x%x%x%x%x%x%x%x%x%x%x")
		if found then
			box.Text = found
			inspect(found)
			flash("ID colado", C.ok)
			return
		end
	end
	box:CaptureFocus()
	flash("Nada válido na área de transferência", C.dim)
end)

fullBtn.MouseButton1Click:Connect(function()
	if busy or closed then return end
	setState("Procurando servidor cheio", C.dim)
	setCount(nil)
	task.spawn(function()
		if closed then return end
		local list = servers(game.PlaceId)
		if closed then return end
		local best, n = nil, -1
		for _, s in ipairs(list) do
			local playing = s.playing or 0
			if s.id ~= game.JobId and playing > n and playing < (s.maxPlayers or 0) then
				best, n = s, playing
			end
		end
		best = best or list[1]
		if best then
			box.Text = best.id
			join(best.id)
		else
			flash("Nenhum servidor disponível", C.bad)
		end
	end)
end)

local function refreshLocal()
	if closed then return end
	local id = strip(box.Text)
	if id == "" or id == game.JobId then
		local p, m = localPlayers()
		setCount(p, m)
	end
end

on(Players.PlayerAdded, refreshLocal)
on(Players.PlayerRemoving, refreshLocal)

on(Teleport.TeleportInitFailed, function(_, result, msg)
	if not busy then return end
	resetJoin()
	flash(tostring(result) .. " " .. tostring(msg):sub(1, 28), C.bad)
end)

-- drag -----------------------------------------------------------------
local dragging, dragStart, startPos = false, nil, nil

head.InputBegan:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		dragStart = i.Position
		startPos = mf.Position
		i.Changed:Connect(function()
			if i.UserInputState == Enum.UserInputState.End then dragging = false end
		end)
	end
end)

on(UIS.InputChanged, function(i)
	if not dragging then return end
	if i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch then
		local d = i.Position - dragStart
		mf.Position = UDim2.new(startPos.X.Scale, startPos.X.Offset + d.X, startPos.Y.Scale, startPos.Y.Offset + d.Y)
	end
end)

-- minimize / close -----------------------------------------------------
local collapsed = false

local function setCollapsed(v)
	collapsed = v
	minBtn.Text = v and "+" or "-"
	body.Visible = not v
	tween(mf, { Size = UDim2.fromOffset(W, v and 44 or H) }, 0.22, Enum.EasingStyle.Quint)
end

minBtn.MouseButton1Click:Connect(function() setCollapsed(not collapsed) end)

local function close()
	if closed then return end
	closed = true
	for i = #conns, 1, -1 do pcall(function() conns[i]:Disconnect() end) end
	pcall(function() getgenv().JobJoinerPro = nil end)
	tween(scale, { Scale = 0.94 }, 0.1)
	task.delay(0.11, function() sg:Destroy() end)
end

closeBtn.MouseButton1Click:Connect(close)

if not isTouch then
	on(UIS.InputBegan, function(i, gp)
		if gp then return end
		if i.KeyCode == Enum.KeyCode.RightShift then
			setCollapsed(not collapsed)
		end
	end)
end

pcall(function()
	getgenv().JobJoinerPro = { close = close }
end)

-- abrir ----------------------------------------------------------------
tween(scale, { Scale = 1 }, 0.18, Enum.EasingStyle.Quint)
inspect("")

if isTouch then
	-- teclado do celular cobre o painel: sobe enquanto digita e volta depois
	local saved
	box.Focused:Connect(function()
		if saved or closed then return end
		saved = mf.Position
		tween(mf, { Position = UDim2.new(saved.X.Scale, saved.X.Offset, 0.3, 0) }, 0.18)
	end)
	box.FocusLost:Connect(function()
		if not saved or closed then return end
		local pos = saved
		saved = nil
		tween(mf, { Position = pos }, 0.18)
	end)
end
