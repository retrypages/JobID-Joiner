--[[
	GLASS TERMINAL - executor mobile (Delta / Android)
	Console Luau com liquid glass, arrastavel, redimensionavel.
	Execute, limpe, pule linha, copie. print/warn vao para o log.
]]

local Players = game:GetService("Players")
local Tween   = game:GetService("TweenService")
local UIS     = game:GetService("UserInputService")

local lp      = Players.LocalPlayer
local isTouch = UIS.TouchEnabled and not UIS.KeyboardEnabled

local C = {
	glass   = Color3.fromRGB(18, 20, 26),
	well    = Color3.fromRGB(8, 9, 13),
	text    = Color3.fromRGB(236, 240, 247),
	dim     = Color3.fromRGB(138, 146, 162),
	faint   = Color3.fromRGB(92, 100, 116),
	line    = Color3.fromRGB(54, 60, 74),
	mint    = Color3.fromRGB(64, 224, 150),
	blue    = Color3.fromRGB(96, 165, 250),
	red     = Color3.fromRGB(255, 95, 86),
	yellow  = Color3.fromRGB(254, 188, 46),
	green   = Color3.fromRGB(40, 200, 64),
}

local TITLE_H = 40

local function mk(class, props, parent)
	local o = Instance.new(class)
	for k, v in next, props do o[k] = v end
	o.Parent = parent
	return o
end

local function tween(o, props, t, style, dir)
	local tw = Tween:Create(o, TweenInfo.new(t or 0.14, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out), props)
	tw:Play()
	return tw
end

local function corner(o, r)
	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(0, r or 12)
	c.Parent = o
	return c
end

local function stroke(o, color, alpha, grad)
	local s = Instance.new("UIStroke")
	s.Color, s.Transparency, s.Thickness = color, alpha or 0.2, 1
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	s.Parent = o
	if grad then
		local g = Instance.new("UIGradient")
		g.Rotation = 90
		g.Color = ColorSequence.new(Color3.fromRGB(255, 255, 255), Color3.fromRGB(120, 130, 150))
		g.Transparency = NumberSequence.new({
			NumberSequenceKeypoint.new(0, 0.25),
			NumberSequenceKeypoint.new(0.55, 0.8),
			NumberSequenceKeypoint.new(1, 0.95),
		})
		g.Parent = s
	end
	return s
end

local conns = {}
local function on(sig, fn)
	local c = sig:Connect(fn)
	conns[#conns + 1] = c
	return c
end

local function tapFx(b, depth)
	local sc = Instance.new("UIScale")
	sc.Parent = b
	b.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			tween(sc, { Scale = depth or 0.95 }, 0.07)
		end
	end)
	b.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			tween(sc, { Scale = 1 }, 0.12, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
		end
	end)
end

local function hud()
	local ok, h = pcall(function() return gethui and gethui() end)
	if ok and h then return h end
	ok, h = pcall(function() return game:GetService("CoreGui") end)
	if ok and h then return h end
	return lp:WaitForChild("PlayerGui")
end

local host = hud()
for _, o in ipairs(host:GetChildren()) do
	if o.Name == "GlassTerminal" and o:IsA("ScreenGui") then o:Destroy() end
end

-- geometria -----------------------------------------------------------------
local cam = workspace.CurrentCamera
local vp  = cam and cam.ViewportSize or Vector2.new(800, 600)

local W = math.min(420, math.max(280, vp.X - 32))
local H = math.min(560, math.max(300, vp.Y - 90))

local sg = mk("ScreenGui", {
	Name = "GlassTerminal",
	ResetOnSpawn = false,
	IgnoreGuiInset = true,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	DisplayOrder = 2147483647,
}, nil)
if not pcall(function() sg.Parent = host end) then
	sg.Parent = lp:WaitForChild("PlayerGui")
end

local mf = mk("Frame", {
	AnchorPoint = Vector2.new(0, 0),
	Position = UDim2.fromOffset((vp.X - W) / 2, (vp.Y - H) / 2 - 12),
	Size = UDim2.fromOffset(W, H),
	BackgroundColor3 = C.glass,
	BackgroundTransparency = 0.22,
	BorderSizePixel = 0,
	ClipsDescendants = false,
	Active = true,
}, sg)
corner(mf, 18)
stroke(mf, C.line, 0.35, true)

pcall(function()
	local sh = Instance.new("UIShadow")
	sh.BlurRadius = UDim.new(0, 26)
	sh.Offset = UDim2.new(0, 0, 0, 10)
	sh.Color = Color3.new(0, 0, 0)
	sh.Transparency = 0.45
	sh.Spread = UDim.new(0, -2)
	sh.Parent = mf
end)

-- vidro: brilho superior + respiro azul embaixo
local sheen = mk("Frame", {
	Size = UDim2.new(1, 0, 1, 0),
	BackgroundColor3 = Color3.fromRGB(255, 255, 255),
	BackgroundTransparency = 0,
	BorderSizePixel = 0,
	ZIndex = 1,
}, mf)
corner(sheen, 18)
local sheenGrad = Instance.new("UIGradient")
sheenGrad.Rotation = 90
sheenGrad.Color = ColorSequence.new(Color3.fromRGB(255, 255, 255), Color3.fromRGB(255, 255, 255))
sheenGrad.Transparency = NumberSequence.new({
	NumberSequenceKeypoint.new(0, 0.72),
	NumberSequenceKeypoint.new(0.16, 0.9),
	NumberSequenceKeypoint.new(0.65, 0.97),
	NumberSequenceKeypoint.new(1, 1),
})
sheenGrad.Parent = sheen

local glow = mk("Frame", {
	Size = UDim2.new(1, 0, 0.45, 0),
	Position = UDim2.new(0, 0, 0.55, 0),
	BackgroundColor3 = C.blue,
	BackgroundTransparency = 0,
	BorderSizePixel = 0,
	ZIndex = 1,
}, mf)
corner(glow, 18)
local glowGrad = Instance.new("UIGradient")
glowGrad.Rotation = 90
glowGrad.Transparency = NumberSequence.new({
	NumberSequenceKeypoint.new(0, 1),
	NumberSequenceKeypoint.new(1, 0.9),
})
glowGrad.Parent = glow

-- titulo --------------------------------------------------------------------
local bar = mk("Frame", {
	Size = UDim2.new(1, 0, 0, TITLE_H),
	BackgroundTransparency = 1,
	ZIndex = 2,
}, mf)

local function light(x, color)
	local b = mk("TextButton", {
		Size = UDim2.fromOffset(12, 12),
		Position = UDim2.fromOffset(x, 14),
		BackgroundColor3 = color,
		BackgroundTransparency = 0.05,
		BorderSizePixel = 0,
		Text = "",
		AutoButtonColor = false,
		ZIndex = 3,
	}, bar)
	corner(b, 999)
	tapFx(b, 0.8)
	if not isTouch then
		b.MouseEnter:Connect(function()
			tween(b, { BackgroundTransparency = 0 }, 0.1)
			tween(b, { Size = UDim2.fromOffset(13, 13) }, 0.1)
		end)
		b.MouseLeave:Connect(function()
			tween(b, { BackgroundTransparency = 0.05 }, 0.1)
			tween(b, { Size = UDim2.fromOffset(12, 12) }, 0.1)
		end)
	end
	return b
end

local closeL = light(14, C.red)
local minL   = light(32, C.yellow)
local maxL   = light(50, C.green)

mk("TextLabel", {
	Size = UDim2.new(1, -160, 1, 0),
	Position = UDim2.fromOffset(70, 0),
	BackgroundTransparency = 1,
	Text = "luau — terminal",
	RichText = true,
	TextColor3 = C.dim,
	TextSize = 11,
	Font = Enum.Font.GothamMedium,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextTruncate = Enum.TextTruncate.AtEnd,
	ZIndex = 3,
}, bar)

local dot = mk("Frame", {
	Size = UDim2.fromOffset(5, 5),
	AnchorPoint = Vector2.new(1, 0.5),
	Position = UDim2.new(1, -64, 0.5, 0),
	BackgroundColor3 = C.mint,
	BackgroundTransparency = 0.15,
	BorderSizePixel = 0,
	ZIndex = 3,
}, bar)
corner(dot, 999)

mk("TextLabel", {
	Size = UDim2.fromOffset(50, 20),
	AnchorPoint = Vector2.new(1, 0.5),
	Position = UDim2.new(1, -14, 0.5, 0),
	BackgroundTransparency = 1,
	Text = "pronto",
	TextColor3 = C.faint,
	TextSize = 10,
	Font = Enum.Font.Code,
	TextXAlignment = Enum.TextXAlignment.Right,
	ZIndex = 3,
}, bar)

-- conteudo ------------------------------------------------------------------
local content = mk("Frame", {
	Size = UDim2.new(1, 0, 1, -TITLE_H),
	Position = UDim2.fromOffset(0, TITLE_H),
	BackgroundTransparency = 1,
	ZIndex = 2,
}, mf)

local toolbar = mk("Frame", {
	Size = UDim2.new(1, -24, 0, 34),
	Position = UDim2.fromOffset(12, 6),
	BackgroundTransparency = 1,
}, content)

local function tool(label, x, w, run)
	local b = mk("TextButton", {
		Size = UDim2.new(w, -6, 1, 0),
		Position = UDim2.new(x, 0, 0, 0),
		BackgroundColor3 = run and C.mint or Color3.fromRGB(38, 42, 52),
		BackgroundTransparency = run and 0 or 0.35,
		BorderSizePixel = 0,
		AutoButtonColor = false,
		RichText = true,
		Text = label,
		TextColor3 = run and Color3.fromRGB(5, 22, 14) or C.text,
		TextSize = 11,
		Font = Enum.Font.GothamBold,
		TextTruncate = Enum.TextTruncate.AtEnd,
	}, toolbar)
	corner(b, 9)
	stroke(b, run and C.mint or C.line, run and 0.5 or 0.55)
	tapFx(b)
	if not isTouch then
		b.MouseEnter:Connect(function()
			tween(b, { BackgroundTransparency = run and 0 or 0.15 }, 0.1)
		end)
		b.MouseLeave:Connect(function()
			tween(b, { BackgroundTransparency = run and 0 or 0.35 }, 0.1)
		end)
	end
	return b
end

local runBtn   = tool('<font color="#0B2A1A">▶</font> Executar', 0.00, 0.33, true)
local clearBtn = tool("Limpar", 0.33, 0.21, false)
local lineBtn  = tool("+ Linha", 0.54, 0.24, false)
local copyBtn  = tool("Copiar", 0.78, 0.22, false)

local well = mk("Frame", {
	Size = UDim2.new(1, -24, 1, -128),
	Position = UDim2.fromOffset(12, 50),
	BackgroundColor3 = C.well,
	BackgroundTransparency = 0.18,
	BorderSizePixel = 0,
}, content)
corner(well, 12)
stroke(well, C.line, 0.5, true)

local wellLine = mk("Frame", {
	Size = UDim2.new(1, -20, 0, 1),
	Position = UDim2.fromOffset(10, 8),
	BackgroundColor3 = C.blue,
	BackgroundTransparency = 0.82,
	BorderSizePixel = 0,
}, well)
corner(wellLine, 999)

local ed = mk("TextBox", {
	Size = UDim2.new(1, -20, 1, -16),
	Position = UDim2.fromOffset(10, 10),
	BackgroundTransparency = 1,
	Text = "",
	PlaceholderText = "-- cole seu script aqui",
	PlaceholderColor3 = C.faint,
	TextColor3 = C.text,
	TextSize = 12,
	Font = Enum.Font.Code,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextYAlignment = Enum.TextYAlignment.Top,
	TextWrapped = true,
	MultiLine = true,
	ClearTextOnFocus = false,
	TextEditable = true,
	ZIndex = 2,
}, well)

local out = mk("TextLabel", {
	Size = UDim2.new(1, -24, 0, 44),
	Position = UDim2.new(0, 12, 1, -72),
	BackgroundTransparency = 1,
	RichText = true,
	Text = '<font color="#5C6474">-- aguardando script</font>',
	TextColor3 = C.dim,
	TextSize = 11,
	Font = Enum.Font.Code,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextYAlignment = Enum.TextYAlignment.Top,
	TextWrapped = false,
	TextTruncate = Enum.TextTruncate.AtEnd,
}, content)

local status = mk("Frame", {
	Size = UDim2.new(1, 0, 0, 22),
	Position = UDim2.new(0, 0, 1, -22),
	BackgroundTransparency = 1,
}, content)

local stLeft = mk("TextLabel", {
	Size = UDim2.new(0.5, -12, 1, 0),
	Position = UDim2.fromOffset(12, 0),
	BackgroundTransparency = 1,
	RichText = true,
	Text = '<font color="#40E096">luau</font> · idle',
	TextColor3 = C.faint,
	TextSize = 10,
	Font = Enum.Font.Code,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextTruncate = Enum.TextTruncate.AtEnd,
}, status)

local stRight = mk("TextLabel", {
	Size = UDim2.new(0.5, -30, 1, 0),
	AnchorPoint = Vector2.new(1, 0),
	Position = UDim2.new(1, -24, 0, 0),
	BackgroundTransparency = 1,
	Text = "0 ln · 0 ch",
	TextColor3 = C.faint,
	TextSize = 10,
	Font = Enum.Font.Code,
	TextXAlignment = Enum.TextXAlignment.Right,
}, status)

-- grip ----------------------------------------------------------------------
local grip = mk("TextButton", {
	Size = UDim2.fromOffset(18, 18),
	AnchorPoint = Vector2.new(1, 1),
	Position = UDim2.new(1, -3, 1, -2),
	BackgroundTransparency = 1,
	Text = "",
	AutoButtonColor = false,
	ZIndex = 6,
}, content)

for i = 1, 3 do
	local l = mk("Frame", {
		Size = UDim2.fromOffset(9 - i * 2, 1),
		AnchorPoint = Vector2.new(1, 1),
		Position = UDim2.new(1, -2, 1, -2 - i * 4),
		BackgroundColor3 = C.faint,
		BackgroundTransparency = 0.25,
		BorderSizePixel = 0,
		Rotation = -45,
	}, grip)
	corner(l, 999)
end

-- helpers -------------------------------------------------------------------
local conns2 = {}
local function track(c) conns2[#conns2 + 1] = c end

local function esc(s)
	return (tostring(s):gsub("&", "&amp;"):gsub("<", "&lt;"):gsub(">", "&gt;"))
end

local logs = {}
local MAX_LOG = 2

local function log(msg, kind)
	local color = kind == "bad" and "#FF9C97" or kind == "ok" and "#7DEFB6" or "#8A92A2"
	local line = esc(msg):gsub("[\r\n\t]+", " ")
	if #line > 130 then line = line:sub(1, 127) .. "..." end
	logs[#logs + 1] = ('<font color="%s">%s</font>'):format(color, line)
	while #logs > MAX_LOG do table.remove(logs, 1) end
	out.Text = table.concat(logs, "\n")
end

local function clip(text)
	local ok = pcall(function() setclipboard(text) end)
	if ok then return true end
	ok = pcall(function() toclipboard(text) end)
	if ok then return true end
	return pcall(function() writeclipboard(text) end)
end

local function updateStatus()
	local text = ed.Text
	local lines = text == "" and 0 or select(2, text:gsub("\n", "")) + 1
	local size = mf.AbsoluteSize
	stRight.Text = ("%d ln · %d ch · %dx%d"):format(lines, #text, math.floor(size.X), math.floor(size.Y))
end

local function run()
	local code = ed.Text
	if not code:find("%S") then
		log("nada para executar", "bad")
		return
	end
	local loader = loadstring or load
	if not loader then
		log("loadstring indisponível neste ambiente", "bad")
		return
	end
	local fn, err = loader(code, "=console")
	if not fn then
		log(err, "bad")
		return
	end

	if setfenv and getfenv then
		pcall(function()
			local base = getfenv(fn)
			local function join(...)
				local t = table.pack(...)
				local o = {}
				for i = 1, t.n do o[i] = tostring(t[i]) end
				return table.concat(o, " ")
			end
			local env = setmetatable({
				print = function(...) log(join(...), "ok") end,
				warn = function(...) log(join(...), "dim") end,
			}, { __index = base, __newindex = base })
			setfenv(fn, env)
		end)
	end

	stLeft.Text = '<font color="#40E096">luau</font> · executando'
	log("executando...", "dim")
	task.spawn(function()
		local ok, res = pcall(fn)
		if ok then
			stLeft.Text = '<font color="#40E096">luau</font> · ok'
			if res ~= nil then log("retorno: " .. tostring(res), "ok") end
		else
			stLeft.Text = '<font color="#FF9C97">luau</font> · erro'
			log(tostring(res), "bad")
		end
	end)
end

-- botoes --------------------------------------------------------------------
runBtn.MouseButton1Click:Connect(run)
clearBtn.MouseButton1Click:Connect(function()
	ed.Text = ""
	logs = {}
	out.Text = '<font color="#5C6474">-- limpo</font>'
	stLeft.Text = '<font color="#40E096">luau</font> · idle'
	updateStatus()
end)
lineBtn.MouseButton1Click:Connect(function()
	ed.Text = ed.Text .. "\n"
	ed:CaptureFocus()
	updateStatus()
end)
copyBtn.MouseButton1Click:Connect(function()
	if ed.Text == "" then
		log("nada para copiar", "dim")
	elseif clip(ed.Text) then
		log("script copiado", "ok")
	else
		log("clipboard indisponível", "bad")
	end
end)
ed:GetPropertyChangedSignal("Text"):Connect(updateStatus)
updateStatus()

if not isTouch then
	on(UIS.InputBegan, function(i, gp)
		if gp then return end
		local ctrl = UIS:IsKeyDown(Enum.KeyCode.LeftControl) or UIS:IsKeyDown(Enum.KeyCode.RightControl)
		if not ctrl then return end
		if i.KeyCode == Enum.KeyCode.Return then
			run()
		elseif i.KeyCode == Enum.KeyCode.L then
			clearBtn:Activate()
		end
	end)
end

-- arrastar ------------------------------------------------------------------
local dragging, dragStart, dragPos = false, nil, nil

bar.InputBegan:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		dragStart = i.Position
		dragPos = mf.Position
		track(i.Changed:Connect(function()
			if i.UserInputState == Enum.UserInputState.End then dragging = false end
		end))
	end
end)

on(UIS.InputChanged, function(i)
	if not dragging then return end
	if i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch then
		local d = i.Position - dragStart
		mf.Position = UDim2.new(dragPos.X.Scale, dragPos.X.Offset + d.X, dragPos.Y.Scale, dragPos.Y.Offset + d.Y)
	end
end)

-- redimensionar -------------------------------------------------------------
local sizing, sizeStart, size0 = false, nil, nil

grip.InputBegan:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
		sizing = true
		sizeStart = i.Position
		size0 = mf.AbsoluteSize
		track(i.Changed:Connect(function()
			if i.UserInputState == Enum.UserInputState.End then sizing = false end
		end))
	end
end)

on(UIS.InputChanged, function(i)
	if not sizing then return end
	if i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch then
		local d = i.Position - sizeStart
		local nx = math.clamp(size0.X + d.X, 280, vp.X - 12)
		local ny = math.clamp(size0.Y + d.Y, 240, vp.Y - 12)
		mf.Size = UDim2.fromOffset(nx, ny)
		updateStatus()
	end
end)

-- janela --------------------------------------------------------------------
local saved = nil
local minimized = false
local miniSize, miniPos = UDim2.fromOffset(W, H), mf.Position

local function maximize()
	if minimized then return end
	if saved then
		mf.Size = saved.size
		mf.Position = saved.pos
		saved = nil
		tween(dot, { BackgroundColor3 = C.mint }, 0.15)
	else
		saved = { size = mf.Size, pos = mf.Position }
		tween(mf, { Size = UDim2.fromOffset(vp.X - 16, vp.Y - 16) }, 0.18, Enum.EasingStyle.Quint)
		tween(mf, { Position = UDim2.fromOffset(8, 8) }, 0.18, Enum.EasingStyle.Quint)
		tween(dot, { BackgroundColor3 = C.blue }, 0.15)
	end
	updateStatus()
end

local function minimize()
	minimized = not minimized
	if minimized then
		miniSize, miniPos = mf.Size, mf.Position
		mf.ClipsDescendants = true
		content.Visible = false
		sheen.Visible = false
		glow.Visible = false
		tween(mf, { Size = UDim2.fromOffset(mf.AbsoluteSize.X, TITLE_H) }, 0.16, Enum.EasingStyle.Quint)
	else
		content.Visible = true
		sheen.Visible = true
		glow.Visible = true
		mf.Position = miniPos
		tween(mf, { Size = miniSize }, 0.18, Enum.EasingStyle.Quint)
		task.delay(0.18, function()
			if not minimized then mf.ClipsDescendants = false end
		end)
	end
end

local closed = false
local function close()
	if closed then return end
	closed = true
	for i = #conns, 1, -1 do pcall(function() conns[i]:Disconnect() end) end
	for i = #conns2, 1, -1 do pcall(function() conns2[i]:Disconnect() end) end
	pcall(function() getgenv().GlassTerminal = nil end)
	local s = mf:FindFirstChildOfClass("UIScale")
	if s then tween(s, { Scale = 0.96 }, 0.1) end
	task.delay(0.11, function() sg:Destroy() end)
end

closeL.MouseButton1Click:Connect(close)
minL.MouseButton1Click:Connect(minimize)
maxL.MouseButton1Click:Connect(maximize)

-- teclado do celular cobre a janela: sobe enquanto digita
if isTouch then
	local raised, home = false, mf.Position
	ed.Focused:Connect(function()
		if raised then return end
		raised = true
		home = mf.Position
		tween(mf, { Position = UDim2.fromOffset(home.X.Offset, 12) }, 0.18)
	end)
	ed.FocusLost:Connect(function()
		if not raised then return end
		raised = false
		tween(mf, { Position = home }, 0.18)
	end)
end

pcall(function()
	getgenv().GlassTerminal = { close = close }
end)

-- abrir ---------------------------------------------------------------------
local sc = Instance.new("UIScale")
sc.Scale = 0.96
sc.Parent = mf
tween(sc, { Scale = 1 }, 0.16, Enum.EasingStyle.Quint)

-- brilho liquido: deriva lenta do gradiente (barato, 1 tween em loop)
Tween:Create(sheenGrad, TweenInfo.new(7, Enum.EasingStyle.Sine, Enum.EasingDirection.InOut, -1, true), {
	Offset = Vector2.new(0, 0.22),
}):Play()
