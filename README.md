--[[
	TERMINAL - executor Luau no estilo Terminal.app dark
	Linhas numeradas, syntax highlight, translucido, arrastavel e redimensionavel.
]]

local Players     = game:GetService("Players")
local Tween       = game:GetService("TweenService")
local UIS         = game:GetService("UserInputService")
local TextService = game:GetService("TextService")

local lp      = Players.LocalPlayer
local isTouch = UIS.TouchEnabled and not UIS.KeyboardEnabled

local C = {
	bg      = Color3.fromRGB(20, 22, 27),
	well    = Color3.fromRGB(9, 10, 13),
	line    = Color3.fromRGB(34, 37, 44),
	text    = Color3.fromRGB(201, 209, 221),
	dim     = Color3.fromRGB(120, 128, 143),
	faint   = Color3.fromRGB(74, 80, 92),
	mint    = Color3.fromRGB(122, 214, 164),
	red     = Color3.fromRGB(224, 108, 117),
	yellow  = Color3.fromRGB(229, 192, 123),
}

local FONT    = Enum.Font.Code
local SIZE    = 12
local TITLE_H = 36
local TOOL_H  = 30

local LH = math.max(14, math.floor(TextService:GetTextSize("A\nA", SIZE, FONT, Vector2.new(4000, 4000)).Y / 2 + 0.5))

local function mk(class, props, parent)
	local o = Instance.new(class)
	for k, v in next, props do o[k] = v end
	o.Parent = parent
	return o
end

local function tween(o, props, t, style, dir)
	local tw = Tween:Create(o, TweenInfo.new(t or 0.12, style or Enum.EasingStyle.Quad, dir or Enum.EasingDirection.Out), props)
	tw:Play()
	return tw
end

local function corner(o, r)
	local c = Instance.new("UICorner")
	c.CornerRadius = UDim.new(0, r or 10)
	c.Parent = o
	return c
end

local function stroke(o, color, alpha)
	local s = Instance.new("UIStroke")
	s.Color, s.Transparency, s.Thickness = color, alpha or 0.3, 1
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	s.Parent = o
	return s
end

local conns = {}
local function on(sig, fn)
	local c = sig:Connect(fn)
	conns[#conns + 1] = c
	return c
end

local function press(b)
	local sc = Instance.new("UIScale")
	sc.Parent = b
	b.InputBegan:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			tween(sc, { Scale = 0.96 }, 0.06)
		end
	end)
	b.InputEnded:Connect(function(i)
		if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
			tween(sc, { Scale = 1 }, 0.1, Enum.EasingStyle.Back, Enum.EasingDirection.Out)
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
	if o.Name == "TerminalLuau" and o:IsA("ScreenGui") then o:Destroy() end
end

local cam = workspace.CurrentCamera
local vp  = cam and cam.ViewportSize or Vector2.new(800, 600)
local W   = math.min(430, math.max(290, vp.X - 28))
local H   = math.min(580, math.max(320, vp.Y - 70))

local sg = mk("ScreenGui", {
	Name = "TerminalLuau",
	ResetOnSpawn = false,
	IgnoreGuiInset = true,
	ZIndexBehavior = Enum.ZIndexBehavior.Sibling,
	DisplayOrder = 2147483647,
}, nil)
if not pcall(function() sg.Parent = host end) then
	sg.Parent = lp:WaitForChild("PlayerGui")
end

local win = mk("Frame", {
	Position = UDim2.fromOffset((vp.X - W) / 2, (vp.Y - H) / 2 - 10),
	Size = UDim2.fromOffset(W, H),
	BackgroundColor3 = C.bg,
	BackgroundTransparency = 0.12,
	BorderSizePixel = 0,
	Active = true,
}, sg)
corner(win, 12)
stroke(win, C.line, 0.25)

pcall(function()
	local sh = Instance.new("UIShadow")
	sh.BlurRadius = UDim.new(0, 24)
	sh.Offset = UDim2.new(0, 0, 0, 8)
	sh.Color = Color3.new(0, 0, 0)
	sh.Transparency = 0.5
	sh.Spread = UDim.new(0, -2)
	sh.Parent = win
end)

-- barra de titulo -------------------------------------------------------------
local bar = mk("Frame", {
	Size = UDim2.new(1, 0, 0, TITLE_H),
	BackgroundTransparency = 1,
}, win)

local function light(x, color)
	local b = mk("TextButton", {
		Size = UDim2.fromOffset(11, 11),
		Position = UDim2.fromOffset(x, 12),
		BackgroundColor3 = color,
		BackgroundTransparency = 0.1,
		BorderSizePixel = 0,
		Text = "",
		AutoButtonColor = false,
	}, bar)
	corner(b, 999)
	press(b)
	if not isTouch then
		b.MouseEnter:Connect(function() tween(b, { BackgroundTransparency = 0 }, 0.1) end)
		b.MouseLeave:Connect(function() tween(b, { BackgroundTransparency = 0.1 }, 0.1) end)
	end
	return b
end

local closeL = light(13, C.red)
local minL   = light(30, C.yellow)
local maxL   = light(47, Color3.fromRGB(90, 190, 110))

mk("TextLabel", {
	Size = UDim2.new(1, -150, 1, 0),
	Position = UDim2.fromOffset(68, 0),
	BackgroundTransparency = 1,
	Text = "luau — untitled.luau",
	TextColor3 = C.dim,
	TextSize = 11,
	Font = Enum.Font.GothamMedium,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextTruncate = Enum.TextTruncate.AtEnd,
}, bar)

mk("Frame", {
	Size = UDim2.new(1, 0, 0, 1),
	Position = UDim2.new(0, 0, 1, -1),
	BackgroundColor3 = C.line,
	BackgroundTransparency = 0.3,
	BorderSizePixel = 0,
}, bar)

-- barra de acoes --------------------------------------------------------------
local tool = mk("Frame", {
	Size = UDim2.new(1, 0, 0, TOOL_H),
	Position = UDim2.fromOffset(0, TITLE_H),
	BackgroundTransparency = 1,
}, win)

local function ghost(label, x, w, color)
	local b = mk("TextButton", {
		Size = UDim2.fromOffset(w, 20),
		Position = UDim2.fromOffset(x, 5),
		BackgroundColor3 = Color3.fromRGB(255, 255, 255),
		BackgroundTransparency = 0.94,
		BorderSizePixel = 0,
		AutoButtonColor = false,
		Text = label,
		TextColor3 = color or C.dim,
		TextSize = 10,
		Font = FONT,
	}, tool)
	corner(b, 5)
	press(b)
	if not isTouch then
		b.MouseEnter:Connect(function() tween(b, { BackgroundTransparency = 0.88, TextColor3 = C.text }, 0.1) end)
		b.MouseLeave:Connect(function() tween(b, { BackgroundTransparency = 0.94, TextColor3 = color or C.dim }, 0.1) end)
	end
	return b
end

local runBtn   = ghost("run", 10, 40, C.mint)
local clearBtn = ghost("clear", 54, 46)
local lineBtn  = ghost("line", 104, 40)
local copyBtn  = ghost("copy", 148, 40)

mk("TextLabel", {
	Size = UDim2.fromOffset(120, 20),
	AnchorPoint = Vector2.new(1, 0),
	Position = UDim2.new(1, -12, 0, 5),
	BackgroundTransparency = 1,
	Text = "luau",
	TextColor3 = C.faint,
	TextSize = 10,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Right,
}, tool)

mk("Frame", {
	Size = UDim2.new(1, 0, 0, 1),
	Position = UDim2.new(0, 0, 1, -1),
	BackgroundColor3 = C.line,
	BackgroundTransparency = 0.30,
	BorderSizePixel = 0,
}, tool)

-- editor ----------------------------------------------------------------------
local well = mk("Frame", {
	Size = UDim2.new(1, -20, 1, -106),
	Position = UDim2.fromOffset(10, TITLE_H + TOOL_H + 10),
	BackgroundColor3 = C.well,
	BackgroundTransparency = 0.1,
	BorderSizePixel = 0,
	ClipsDescendants = true,
}, win)
corner(well, 8)
stroke(well, C.line, 0.35)

local scroll = mk("ScrollingFrame", {
	Size = UDim2.new(1, -16, 1, -68),
	Position = UDim2.fromOffset(8, 6),
	BackgroundTransparency = 1,
	BorderSizePixel = 0,
	ScrollBarThickness = 3,
	ScrollBarImageColor3 = C.faint,
	ScrollBarImageTransparency = 0.4,
	ScrollingDirection = Enum.ScrollingDirection.Y,
	CanvasSize = UDim2.new(0, 0, 0, 0),
	ClipsDescendants = true,
}, well)

local view = mk("TextLabel", {
	Size = UDim2.new(0, 100, 0, 20),
	Position = UDim2.fromOffset(40, 0),
	BackgroundTransparency = 1,
	RichText = true,
	Text = "",
	TextColor3 = C.text,
	TextSize = SIZE,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextYAlignment = Enum.TextYAlignment.Top,
	TextWrapped = false,
	ZIndex = 1,
}, scroll)

local gutter = mk("TextLabel", {
	Size = UDim2.new(0, 30, 0, 20),
	BackgroundTransparency = 1,
	Text = "1",
	TextColor3 = C.faint,
	TextSize = SIZE,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Right,
	TextYAlignment = Enum.TextYAlignment.Top,
	TextWrapped = false,
	ZIndex = 2,
}, scroll)

local ed = mk("TextBox", {
	Size = UDim2.new(0, 100, 0, 20),
	Position = UDim2.fromOffset(40, 0),
	BackgroundTransparency = 1,
	Text = "",
	TextColor3 = C.text,
	TextSize = SIZE,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextYAlignment = Enum.TextYAlignment.Top,
	TextWrapped = false,
	MultiLine = true,
	ClearTextOnFocus = false,
	ZIndex = 2,
}, scroll)

-- camada de toque: permite rolar com o dedo e focar com um toque
local hit = mk("TextButton", {
	Size = UDim2.new(0, 100, 0, 20),
	Position = UDim2.fromOffset(40, 0),
	BackgroundTransparency = 1,
	Text = "",
	AutoButtonColor = false,
	ZIndex = 4,
}, scroll)

local outLine = mk("Frame", {
	Size = UDim2.new(1, 0, 0, 1),
	Position = UDim2.new(0, 0, 1, -54),
	BackgroundColor3 = C.line,
	BackgroundTransparency = 0.35,
	BorderSizePixel = 0,
}, well)

local out = mk("TextLabel", {
	Size = UDim2.new(1, -16, 0, 44),
	Position = UDim2.new(0, 8, 1, -50),
	BackgroundTransparency = 1,
	RichText = true,
	Text = "",
	TextColor3 = C.dim,
	TextSize = 11,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextYAlignment = Enum.TextYAlignment.Top,
	TextWrapped = false,
	TextTruncate = Enum.TextTruncate.AtEnd,
}, well)

-- barra de estado -------------------------------------------------------------
local status = mk("Frame", {
	Size = UDim2.new(1, 0, 0, 22),
	Position = UDim2.new(0, 0, 1, -24),
	BackgroundTransparency = 1,
}, win)

local stLeft = mk("TextLabel", {
	Size = UDim2.new(0.5, -12, 1, 0),
	Position = UDim2.fromOffset(12, 0),
	BackgroundTransparency = 1,
	RichText = true,
	Text = '<font color="#7AD6A4">luau</font>  pronto',
	TextColor3 = C.faint,
	TextSize = 10,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextTruncate = Enum.TextTruncate.AtEnd,
}, status)

local stRight = mk("TextLabel", {
	Size = UDim2.new(0.5, -34, 1, 0),
	AnchorPoint = Vector2.new(1, 0),
	Position = UDim2.new(1, -30, 0, 0),
	BackgroundTransparency = 1,
	Text = "1 linha · 0 chars",
	TextColor3 = C.faint,
	TextSize = 10,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Right,
}, status)

local grip = mk("TextButton", {
	Size = UDim2.fromOffset(16, 16),
	AnchorPoint = Vector2.new(1, 1),
	Position = UDim2.new(1, -5, 1, -4),
	BackgroundTransparency = 1,
	Text = "",
	AutoButtonColor = false,
	ZIndex = 6,
}, win)

for i = 1, 3 do
	local l = mk("Frame", {
		Size = UDim2.fromOffset(8 - i, 1),
		AnchorPoint = Vector2.new(1, 1),
		Position = UDim2.new(1, 0, 1, -i * 4),
		BackgroundColor3 = C.faint,
		BackgroundTransparency = 0.2,
		BorderSizePixel = 0,
		Rotation = -45,
	}, grip)
	corner(l, 999)
end

-- syntax highlight ------------------------------------------------------------
local KW = {}
for w in ("local function end if then else elseif for while do return and or not nil true false in repeat until break continue export type"):gmatch("%S+") do
	KW[w] = true
end

local BI = {}
for w in ("game workspace script print warn task wait spawn delay pcall xpcall loadstring require getgenv setclipboard pairs ipairs next select type typeof tostring tonumber error math table string os coroutine Instance Enum Vector2 Vector3 CFrame Color3 UDim2 UDim TextService TweenInfo"):gmatch("%S+") do
	BI[w] = true
end

local CL = {
	kw   = "#C3A6E8",
	str  = "#9ECE6A",
	com  = "#5A6270",
	num  = "#E5C07B",
	fn   = "#7FB4F5",
	bi   = "#73C2D8",
	punc = "#7C8596",
	base = "#C9D1DD",
}

local function esc(s)
	return (tostring(s):gsub("&", "&amp;"):gsub("<", "&lt;"):gsub(">", "&gt;"))
end

local function paint(color, text)
	return ('<font color="%s">%s</font>'):format(color, esc(text))
end

local function hl(src)
	if src == "" then
		return '<font color="#4A505C">-- cole seu script aqui</font>'
	end
	local buf = {}
	local i, n = 1, #src
	while i <= n do
		local c = src:sub(i, i)
		local d = src:sub(i + 1, i + 1)

		if c == "-" and d == "-" then
			local j = src:find("\n", i, true) or (n + 1)
			buf[#buf + 1] = paint(CL.com, src:sub(i, j - 1))
			i = j
		elseif c == '"' or c == "'" then
			local j = i + 1
			while j <= n do
				local ch = src:sub(j, j)
				if ch == "\\" then
					j = j + 2
				elseif ch == c then
					j = j + 1
					break
				elseif ch == "\n" then
					break
				else
					j = j + 1
				end
			end
			buf[#buf + 1] = paint(CL.str, src:sub(i, j - 1))
			i = j
		elseif c:match("[%a_]") then
			local j = i
			while j <= n and src:sub(j, j):match("[%w_]") do j = j + 1 end
			local word = src:sub(i, j - 1)
			if KW[word] then
				buf[#buf + 1] = paint(CL.kw, word)
			elseif BI[word] then
				buf[#buf + 1] = paint(CL.bi, word)
			else
				local k = j
				while k <= n and src:sub(k, k):match("[ \t]") do k = k + 1 end
				buf[#buf + 1] = paint(src:sub(k, k) == "(" and CL.fn or CL.base, word)
			end
			i = j
		elseif c:match("%d") then
			local j = i
			while j <= n and src:sub(j, j):match("[%w_%.]") do j = j + 1 end
			buf[#buf + 1] = paint(CL.num, src:sub(i, j - 1))
			i = j
		else
			local j = i
			while j <= n do
				local ch = src:sub(j, j)
				if ch:match("[%a_]") or ch:match("%d") or ch == "-" or ch == '"' or ch == "'" then break end
				j = j + 1
			end
			if j == i then j = i + 1 end
			buf[#buf + 1] = paint(CL.punc, src:sub(i, j - 1))
			i = j
		end
	end
	return table.concat(buf)
end

-- estado ----------------------------------------------------------------------
local logs = {}
local state = "pronto"

local function log(msg, kind)
	local color = kind == "bad" and "#E06C75" or kind == "ok" and "#98C379" or "#5C6370"
	local text = esc(msg):gsub("[\r\n\t]+", " ")
	if #text > 150 then text = text:sub(1, 147) .. "..." end
	logs[#logs + 1] = ('<font color="#5A6270">›</font> <font color="%s">%s</font>'):format(color, text)
	while #logs > 3 do table.remove(logs, 1) end
	out.Text = table.concat(logs, "\n")
end

local function setState(label, color)
	state = label
	stLeft.Text = ('<font color="#7AD6A4">luau</font>  <font color="%s">%s</font>'):format(color or "#4A505C", label)
end

local cursorLine = nil
pcall(function()
	cursorLine = function()
		local ok, pos = pcall(function() return ed.CursorPosition end)
		if not ok or type(pos) ~= "number" or pos < 0 then return nil end
		return select(2, ed.Text:sub(1, pos):gsub("\n", "")) + 1
	end
end)

local function layout()
	local text = ed.Text
	local lines = text == "" and 1 or select(2, text:gsub("\n", "")) + 1
	local h = math.max(lines * LH, LH)
	local w = math.max(scroll.AbsoluteSize.X, 120)

	scroll.CanvasSize = UDim2.new(0, w, 0, h + 12)
	gutter.Size = UDim2.new(0, 30, 0, h)
	ed.Size = UDim2.new(0, w - 46, 0, h)
	view.Size = ed.Size
	hit.Size = ed.Size

	local nums = table.create and table.create(lines) or {}
	for i = 1, lines do nums[i] = i end
	gutter.Text = table.concat(nums, "\n")

	local plural = lines == 1 and "linha" or "linhas"
	stRight.Text = ("%d %s · %d chars · %dx%d"):format(lines, plural, #text, math.floor(win.AbsoluteSize.X), math.floor(win.AbsoluteSize.Y))
end

local function follow()
	if not cursorLine then return end
	local line = cursorLine()
	if not line then return end
	local y = (line - 1) * LH
	local top = scroll.CanvasPosition.Y
	local viewH = scroll.AbsoluteSize.Y
	if y < top then
		scroll.CanvasPosition = Vector2.new(0, math.max(0, y - LH))
	elseif y + LH > top + viewH then
		scroll.CanvasPosition = Vector2.new(0, y + LH - viewH + LH)
	end
end

local function render()
	view.Text = hl(ed.Text)
end

local function run()
	local code = ed.Text
	if not code:find("%S") then
		log("nada para executar", "bad")
		return
	end
	local loader = loadstring or load
	if not loader then
		log("loadstring indisponivel neste ambiente", "bad")
		return
	end
	local fn, err = loader(code, "=console")
	if not fn then
		setState("erro de sintaxe", "#E06C75")
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
			setfenv(fn, setmetatable({
				print = function(...) log(join(...), "ok") end,
				warn = function(...) log(join(...), "dim") end,
			}, { __index = base, __newindex = base }))
		end)
	end

	setState("executando", "#E5C07B")
	log("executando script", "dim")
	task.spawn(function()
		local ok, res = pcall(fn)
		if ok then
			setState("ok", "#7AD6A4")
			if res ~= nil then log("retorno: " .. tostring(res), "ok") end
		else
			setState("erro", "#E06C75")
			log(tostring(res), "bad")
		end
	end)
end

-- acoes -----------------------------------------------------------------------
runBtn.MouseButton1Click:Connect(run)

clearBtn.MouseButton1Click:Connect(function()
	ed.Text = ""
	logs = {}
	out.Text = ""
	setState("pronto")
	layout()
	render()
end)

lineBtn.MouseButton1Click:Connect(function()
	ed.Text = ed.Text .. "\n"
	ed:CaptureFocus()
	layout()
	task.defer(follow)
end)

copyBtn.MouseButton1Click:Connect(function()
	if ed.Text == "" then
		log("nada para copiar", "dim")
		return
	end
	local ok = pcall(function() setclipboard(ed.Text) end)
	if not ok then ok = pcall(function() toclipboard(ed.Text) end) end
	if not ok then ok = pcall(function() writeclipboard(ed.Text) end) end
	log(ok and "script copiado" or "clipboard indisponivel", ok and "ok" or "bad")
end)

ed:GetPropertyChangedSignal("Text"):Connect(function()
	layout()
	if ed:IsFocused() then follow() end
end)

ed.Focused:Connect(function()
	ed.TextTransparency = 0
	ed.ZIndex = 3
	view.ZIndex = 1
	hit.Visible = false
	setState("editando", "#7AD6A4")
end)

ed.FocusLost:Connect(function(enter)
	ed.TextTransparency = 1
	ed.ZIndex = 2
	render()
	view.ZIndex = 3
	hit.Visible = true
	setState("pronto")
	if enter then run() end
end)

hit.Activated:Connect(function()
	ed:CaptureFocus()
end)

if not isTouch then
	on(UIS.InputBegan, function(i, gp)
		if gp then return end
		if i.KeyCode == Enum.KeyCode.Return and UIS:IsKeyDown(Enum.KeyCode.LeftControl) then
			run()
		elseif i.KeyCode == Enum.KeyCode.L and UIS:IsKeyDown(Enum.KeyCode.LeftControl) then
			clearBtn:Activate()
		end
	end)
end

-- arrastar --------------------------------------------------------------------
local dragging, dragStart, dragPos = false, nil, nil

bar.InputBegan:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		dragStart = i.Position
		dragPos = win.Position
		i.Changed:Connect(function()
			if i.UserInputState == Enum.UserInputState.End then dragging = false end
		end)
	end
end)

on(UIS.InputChanged, function(i)
	if not dragging then return end
	if i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch then
		local d = i.Position - dragStart
		win.Position = UDim2.fromOffset(dragPos.X.Offset + d.X, dragPos.Y.Offset + d.Y)
	end
end)

-- redimensionar ---------------------------------------------------------------
local sizing, sizeStart, size0 = false, nil, nil

grip.InputBegan:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
		sizing = true
		sizeStart = i.Position
		size0 = win.AbsoluteSize
		i.Changed:Connect(function()
			if i.UserInputState == Enum.UserInputState.End then sizing = false end
		end)
	end
end)

on(UIS.InputChanged, function(i)
	if not sizing then return end
	if i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch then
		local d = i.Position - sizeStart
		win.Size = UDim2.fromOffset(
			math.clamp(size0.X + d.X, 290, vp.X - 10),
			math.clamp(size0.Y + d.Y, 300, vp.Y - 10)
		)
		layout()
	end
end)

-- janela ----------------------------------------------------------------------
local saved, miniSize, miniPos = nil, UDim2.fromOffset(W, H), win.Position
local minimized = false

local function maximize()
	if minimized then return end
	if saved then
		win.Size, win.Position = saved.size, saved.pos
		saved = nil
	else
		saved = { size = win.Size, pos = win.Position }
		tween(win, { Size = UDim2.fromOffset(vp.X - 14, vp.Y - 14) }, 0.16, Enum.EasingStyle.Quint)
		tween(win, { Position = UDim2.fromOffset(7, 7) }, 0.16, Enum.EasingStyle.Quint)
	end
	layout()
end

local function minimize()
	minimized = not minimized
	if minimized then
		miniSize, miniPos = win.Size, win.Position
		tool.Visible = false
		well.Visible = false
		status.Visible = false
		win.ClipsDescendants = true
		tween(win, { Size = UDim2.fromOffset(win.AbsoluteSize.X, TITLE_H) }, 0.15, Enum.EasingStyle.Quint)
	else
		tool.Visible = true
		well.Visible = true
		status.Visible = true
		win.Position = miniPos
		tween(win, { Size = miniSize }, 0.16, Enum.EasingStyle.Quint)
		task.delay(0.16, function()
			if not minimized then win.ClipsDescendants = false end
		end)
	end
end

local closed = false
local function close()
	if closed then return end
	closed = true
	for i = #conns, 1, -1 do pcall(function() conns[i]:Disconnect() end) end
	pcall(function() getgenv().TerminalLuau = nil end)
	local sc = win:FindFirstChildOfClass("UIScale")
	if sc then tween(sc, { Scale = 0.97 }, 0.09) end
	task.delay(0.1, function() sg:Destroy() end)
end

closeL.MouseButton1Click:Connect(close)
minL.MouseButton1Click:Connect(minimize)
maxL.MouseButton1Click:Connect(maximize)

if isTouch then
	local raised, home = false, win.Position
	ed.Focused:Connect(function()
		if raised then return end
		raised = true
		home = win.Position
		tween(win, { Position = UDim2.fromOffset(home.X.Offset, 10) }, 0.16)
	end)
	ed.FocusLost:Connect(function()
		if not raised then return end
		raised = false
		tween(win, { Position = home }, 0.16)
	end)
end

pcall(function()
	getgenv().TerminalLuau = { close = close }
end)

-- abrir -----------------------------------------------------------------------
local sc = Instance.new("UIScale")
sc.Scale = 0.97
sc.Parent = win
tween(sc, { Scale = 1 }, 0.14, Enum.EasingStyle.Quint)

ed.TextTransparency = 1
view.ZIndex = 3
hit.Visible = true
layout()
render()
task.defer(layout)
