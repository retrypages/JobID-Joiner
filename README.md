--[[
	TERMINAL - executor Luau, chrome de Terminal.app dark com liquid glass.
	Linhas, syntax highlight, temas, tema atual na linha, erro na linha, glass em tudo.
]]

local Players     = game:GetService("Players")
local Tween       = game:GetService("TweenService")
local UIS         = game:GetService("UserInputService")
local TextService = game:GetService("TextService")
local RunService  = game:GetService("RunService")

local lp      = Players.LocalPlayer
local isTouch = UIS.TouchEnabled and not UIS.KeyboardEnabled

local FONT = Enum.Font.RobotoMono
local SIZE = 12
local LH   = 16

local THEMES = {
	{
		name = "void",
		bg    = Color3.fromRGB(17, 19, 24),
		panel = Color3.fromRGB(13, 15, 19),
		well  = Color3.fromRGB(8, 9, 12),
		line  = Color3.fromRGB(34, 37, 44),
		text  = Color3.fromRGB(201, 209, 221),
		dim   = Color3.fromRGB(122, 130, 145),
		faint = Color3.fromRGB(78, 84, 96),
		accent= Color3.fromRGB(122, 214, 164),
		red   = Color3.fromRGB(224, 108, 117),
		num   = Color3.fromRGB(229, 192, 123),
		kw    = "#C3A6E8", str = "#9ECE6A", com = "#5A6270", fnc = "#7FB4F5",
		bi    = "#73C2D8", punc = "#7C8596", base = "#C9D1DD",
	},
	{
		name = "graphite",
		bg    = Color3.fromRGB(24, 25, 29),
		panel = Color3.fromRGB(18, 19, 23),
		well  = Color3.fromRGB(10, 11, 14),
		line  = Color3.fromRGB(41, 43, 51),
		text  = Color3.fromRGB(206, 211, 222),
		dim   = Color3.fromRGB(126, 132, 146),
		faint = Color3.fromRGB(80, 85, 97),
		accent= Color3.fromRGB(155, 184, 255),
		red   = Color3.fromRGB(235, 120, 130),
		num   = Color3.fromRGB(214, 188, 138),
		kw    = "#AFB8FF", str = "#9EDB9A", com = "#5B6270", fnc = "#9BB8FF",
		bi    = "#82CFE0", punc = "#7E8798", base = "#CED5E0",
	},
	{
		name = "amber",
		bg    = Color3.fromRGB(26, 23, 19),
		panel = Color3.fromRGB(20, 17, 13),
		well  = Color3.fromRGB(11, 9, 7),
		line  = Color3.fromRGB(45, 39, 31),
		text  = Color3.fromRGB(226, 215, 199),
		dim   = Color3.fromRGB(150, 137, 119),
		faint = Color3.fromRGB(96, 86, 72),
		accent= Color3.fromRGB(229, 181, 103),
		red   = Color3.fromRGB(230, 120, 110),
		num   = Color3.fromRGB(231, 192, 122),
		kw    = "#DDBB92", str = "#A8C48A", com = "#6A5F52", fnc = "#E0B070",
		bi    = "#C9A87B", punc = "#8A7C6B", base = "#DED2C2",
	},
}

local T = THEMES[1]

local function hex(c)
	return ("#%02X%02X%02X"):format(math.floor(c.R * 255), math.floor(c.G * 255), math.floor(c.B * 255))
end

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

local function stroke(o, alpha)
	local s = Instance.new("UIStroke")
	s.Color, s.Transparency, s.Thickness = T.line, alpha or 0.25, 1
	s.ApplyStrokeMode = Enum.ApplyStrokeMode.Border
	s.Parent = o
	return s
end

local binds = {}
local function bind(obj, prop, key)
	binds[#binds + 1] = { obj, prop, key }
end

local function hairline(parent, y, off)
	local f = mk("Frame", {
		Size = UDim2.new(1, 0, 0, 1),
		Position = UDim2.new(0, 0, y, off),
		BackgroundColor3 = T.line,
		BackgroundTransparency = 0.15,
		BorderSizePixel = 0,
		ZIndex = 7,
	}, parent)
	local g = Instance.new("UIGradient")
	g.Transparency = NumberSequence.new({
		NumberSequenceKeypoint.new(0, 1),
		NumberSequenceKeypoint.new(0.1, 0.35),
		NumberSequenceKeypoint.new(0.9, 0.35),
		NumberSequenceKeypoint.new(1, 1),
	})
	g.Parent = f
	bind(f, "BackgroundColor3", "line")
	return f
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
			tween(sc, { Scale = 0.95 }, 0.06)
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
local W   = math.min(430, math.max(300, vp.X - 28))
local H   = math.min(600, math.max(330, vp.Y - 70))

if cam then
	on(cam:GetPropertyChangedSignal("ViewportSize"), function()
		vp = cam.ViewportSize
	end)
end

local TITLE_H, TAB_H, TOOL_H, STAT_H = 34, 25, 30, 20

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

-- sombra em camadas -----------------------------------------------------------
local shadows = {}
for i = 1, 3 do
	local s = mk("Frame", {
		BackgroundColor3 = Color3.new(0, 0, 0),
		BackgroundTransparency = 0.9 + i * 0.025,
		BorderSizePixel = 0,
		ZIndex = 0,
	}, sg)
	corner(s, 12 + i * 5)
	shadows[i] = s
end

local win = mk("Frame", {
	Position = UDim2.fromOffset((vp.X - W) / 2, (vp.Y - H) / 2 - 10),
	Size = UDim2.fromOffset(W, H),
	BackgroundColor3 = T.bg,
	BackgroundTransparency = 0.1,
	BorderSizePixel = 0,
	Active = true,
	ZIndex = 1,
}, sg)
corner(win, 12)
local winStroke = stroke(win, 0.2)
bind(win, "BackgroundColor3", "bg")
bind(winStroke, "Color", "line")

local sheen = mk("Frame", {
	Size = UDim2.new(1, 0, 1, 0),
	BackgroundColor3 = Color3.fromRGB(255, 255, 255),
	BorderSizePixel = 0,
	ZIndex = 2,
}, win)
corner(sheen, 12)
local sheenGrad = Instance.new("UIGradient")
sheenGrad.Rotation = 90
sheenGrad.Transparency = NumberSequence.new({
	NumberSequenceKeypoint.new(0, 0.83),
	NumberSequenceKeypoint.new(0.14, 0.96),
	NumberSequenceKeypoint.new(1, 1),
})
sheenGrad.Parent = sheen

local topGlow = mk("Frame", {
	Size = UDim2.new(1, -24, 0, 1),
	Position = UDim2.fromOffset(12, 0),
	BackgroundColor3 = Color3.fromRGB(255, 255, 255),
	BackgroundTransparency = 0.82,
	BorderSizePixel = 0,
	ZIndex = 3,
}, win)
corner(topGlow, 999)

local syncShadow
do
	local function place()
		local px, py = win.Position.X.Offset, win.Position.Y.Offset
		local sx, sy = win.AbsoluteSize.X, win.AbsoluteSize.Y
		for i = 1, 3 do
			local pad = i * 5
			local s = shadows[i]
			s.Position = UDim2.fromOffset(px - pad, py - pad + i * 3)
			s.Size = UDim2.fromOffset(sx + pad * 2, sy + pad * 2)
		end
	end
	syncShadow = place
end

local lastX, lastY, lastW, lastH
on(RunService.RenderStepped, function()
	local x, y = win.Position.X.Offset, win.Position.Y.Offset
	local w, h = win.AbsoluteSize.X, win.AbsoluteSize.Y
	if x ~= lastX or y ~= lastY or w ~= lastW or h ~= lastH then
		lastX, lastY, lastW, lastH = x, y, w, h
		syncShadow()
	end
end)

-- titulo ----------------------------------------------------------------------
local bar = mk("Frame", {
	Size = UDim2.new(1, 0, 0, TITLE_H),
	BackgroundTransparency = 1,
	ZIndex = 4,
}, win)

local function light(x, color)
	local b = mk("TextButton", {
		Size = UDim2.fromOffset(11, 11),
		Position = UDim2.fromOffset(x, 11),
		BackgroundColor3 = color,
		BackgroundTransparency = 0.08,
		BorderSizePixel = 0,
		Text = "",
		AutoButtonColor = false,
		ZIndex = 5,
	}, bar)
	corner(b, 999)
	press(b)
	if not isTouch then
		b.MouseEnter:Connect(function() tween(b, { BackgroundTransparency = 0 }, 0.1) end)
		b.MouseLeave:Connect(function() tween(b, { BackgroundTransparency = 0.08 }, 0.1) end)
	end
	return b
end

local closeL = light(13, T.red)
local minL   = light(30, T.num)
local maxL   = light(47, Color3.fromRGB(90, 190, 110))

local title = mk("TextLabel", {
	Size = UDim2.new(1, -150, 1, 0),
	Position = UDim2.fromOffset(68, 0),
	BackgroundTransparency = 1,
	Text = "luau — terminal",
	TextColor3 = T.dim,
	TextSize = 11,
	Font = Enum.Font.GothamMedium,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextTruncate = Enum.TextTruncate.AtEnd,
	ZIndex = 5,
}, bar)
bind(title, "TextColor3", "dim")

-- barra de abas ---------------------------------------------------------------
local tabs = mk("Frame", {
	Size = UDim2.new(1, 0, 0, TAB_H),
	Position = UDim2.fromOffset(0, TITLE_H),
	BackgroundColor3 = T.panel,
	BackgroundTransparency = 0.55,
	BorderSizePixel = 0,
	ZIndex = 4,
}, win)
bind(tabs, "BackgroundColor3", "panel")
hairline(tabs, 1, -1)

local tab = mk("TextButton", {
	Size = UDim2.fromOffset(128, 18),
	Position = UDim2.fromOffset(9, 3),
	BackgroundColor3 = T.bg,
	BackgroundTransparency = 0.3,
	BorderSizePixel = 0,
	Text = "",
	AutoButtonColor = false,
	ZIndex = 5,
}, tabs)
corner(tab, 5)
bind(tab, "BackgroundColor3", "bg")

local tabLabel = mk("TextLabel", {
	Size = UDim2.new(1, -22, 1, 0),
	Position = UDim2.fromOffset(8, 0),
	BackgroundTransparency = 1,
	Text = "untitled.luau",
	TextColor3 = T.text,
	TextSize = 10,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Left,
	ZIndex = 6,
}, tab)
bind(tabLabel, "TextColor3", "text")

local tabDot = mk("Frame", {
	Size = UDim2.fromOffset(4, 4),
	AnchorPoint = Vector2.new(1, 0.5),
	Position = UDim2.new(1, -7, 0.5, 0),
	BackgroundColor3 = T.accent,
	BorderSizePixel = 0,
	ZIndex = 6,
}, tab)
corner(tabDot, 999)
bind(tabDot, "BackgroundColor3", "accent")

-- barra de acoes --------------------------------------------------------------
local tool = mk("Frame", {
	Size = UDim2.new(1, 0, 0, TOOL_H),
	Position = UDim2.fromOffset(0, TITLE_H + TAB_H),
	BackgroundTransparency = 1,
	ZIndex = 4,
}, win)
hairline(tool, 1, -1)

local function ghost(label, x, w, color, parent)
	local b = mk("TextButton", {
		Size = UDim2.fromOffset(w, 20),
		Position = UDim2.fromOffset(x, 5),
		BackgroundColor3 = Color3.fromRGB(255, 255, 255),
		BackgroundTransparency = 0.94,
		BorderSizePixel = 0,
		AutoButtonColor = false,
		Text = label,
		TextColor3 = color or T.dim,
		TextSize = 10,
		Font = FONT,
		ZIndex = 5,
	}, parent or tool)
	corner(b, 5)
	press(b)
	if not isTouch then
		b.MouseEnter:Connect(function() tween(b, { BackgroundTransparency = 0.87 }, 0.1) end)
		b.MouseLeave:Connect(function() tween(b, { BackgroundTransparency = 0.94 }, 0.1) end)
	end
	return b
end

local runBtn   = ghost("run", 9, 38, T.accent)
local clearBtn = ghost("clear", 51, 44)
local lineBtn  = ghost("line", 99, 38)
local copyBtn  = ghost("copy", 141, 38)
bind(runBtn, "TextColor3", "accent")
bind(clearBtn, "TextColor3", "dim")
bind(lineBtn, "TextColor3", "dim")
bind(copyBtn, "TextColor3", "dim")

local themeBtn = ghost(T.name, 0, 44, T.dim)
themeBtn.AnchorPoint = Vector2.new(1, 0)
themeBtn.Position = UDim2.new(1, -10, 0, 5)
bind(themeBtn, "TextColor3", "dim")

local bigBtn = ghost("A+", 0, 18, T.dim)
bigBtn.AnchorPoint = Vector2.new(1, 0)
bigBtn.Position = UDim2.new(1, -56, 0, 5)
bigBtn.TextSize = 9
bind(bigBtn, "TextColor3", "dim")

local smallBtn = ghost("A-", 0, 18, T.dim)
smallBtn.AnchorPoint = Vector2.new(1, 0)
smallBtn.Position = UDim2.new(1, -76, 0, 5)
smallBtn.TextSize = 9
bind(smallBtn, "TextColor3", "dim")

local shimmer = mk("Frame", {
	Size = UDim2.new(0.4, 0, 0, 1),
	Position = UDim2.new(-0.4, 0, 1, -1),
	BackgroundColor3 = T.accent,
	BorderSizePixel = 0,
	Visible = false,
	ZIndex = 6,
}, tool)
bind(shimmer, "BackgroundColor3", "accent")
local shimmerGrad = Instance.new("UIGradient")
shimmerGrad.Transparency = NumberSequence.new({
	NumberSequenceKeypoint.new(0, 1),
	NumberSequenceKeypoint.new(0.5, 0.15),
	NumberSequenceKeypoint.new(1, 1),
})
shimmerGrad.Parent = shimmer
local shimmerTween

-- editor ----------------------------------------------------------------------
local well = mk("Frame", {
	Size = UDim2.new(1, -20, 1, -125),
	Position = UDim2.fromOffset(10, TITLE_H + TAB_H + TOOL_H + 8),
	BackgroundColor3 = T.well,
	BackgroundTransparency = 0.12,
	BorderSizePixel = 0,
	ClipsDescendants = true,
	ZIndex = 4,
}, win)
corner(well, 8)
stroke(well, 0.35)
bind(well, "BackgroundColor3", "well")

local scroll = mk("ScrollingFrame", {
	Size = UDim2.new(1, -16, 1, -68),
	Position = UDim2.fromOffset(8, 6),
	BackgroundTransparency = 1,
	BorderSizePixel = 0,
	ScrollBarThickness = 3,
	ScrollBarImageColor3 = T.faint,
	ScrollBarImageTransparency = 0.3,
	ScrollingDirection = Enum.ScrollingDirection.Y,
	CanvasSize = UDim2.new(0, 0, 0, 0),
	ClipsDescendants = true,
	ZIndex = 5,
}, well)

local empty = mk("TextLabel", {
	Size = UDim2.new(1, 0, 1, -60),
	BackgroundTransparency = 1,
	Text = "λ",
	TextColor3 = T.line,
	TextSize = 60,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Center,
	TextYAlignment = Enum.TextYAlignment.Center,
	ZIndex = 4,
}, well)
bind(empty, "TextColor3", "line")

local emptyHint = mk("TextLabel", {
	Size = UDim2.new(1, 0, 0, 16),
	Position = UDim2.new(0, 0, 1, -84),
	BackgroundTransparency = 1,
	Text = "-- run para comecar",
	TextColor3 = T.faint,
	TextSize = 10,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Center,
	ZIndex = 4,
}, well)
bind(emptyHint, "TextColor3", "faint")

local fadeTop = mk("Frame", {
	Size = UDim2.new(1, 0, 0, 14),
	Position = UDim2.fromOffset(0, 6),
	BackgroundColor3 = T.well,
	BorderSizePixel = 0,
	ZIndex = 14,
}, well)
bind(fadeTop, "BackgroundColor3", "well")
local ft = Instance.new("UIGradient")
ft.Rotation = 90
ft.Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 0.05), NumberSequenceKeypoint.new(1, 1) })
ft.Parent = fadeTop

local fadeBottom = mk("Frame", {
	Size = UDim2.new(1, 0, 0, 14),
	Position = UDim2.new(0, 0, 1, -76),
	BackgroundColor3 = T.well,
	BorderSizePixel = 0,
	ZIndex = 14,
}, well)
bind(fadeBottom, "BackgroundColor3", "well")
local fb = Instance.new("UIGradient")
fb.Rotation = 90
fb.Transparency = NumberSequence.new({ NumberSequenceKeypoint.new(0, 1), NumberSequenceKeypoint.new(1, 0.05) })
fb.Parent = fadeBottom

local errLine = mk("Frame", {
	Size = UDim2.new(1, 0, 0, 16),
	BackgroundColor3 = T.red,
	BackgroundTransparency = 0.9,
	BorderSizePixel = 0,
	Visible = false,
	ZIndex = 6,
}, scroll)
bind(errLine, "BackgroundColor3", "red")

local curLine = mk("Frame", {
	Size = UDim2.new(1, 0, 0, LH),
	BackgroundColor3 = Color3.new(1, 1, 1),
	BackgroundTransparency = 0.955,
	BorderSizePixel = 0,
	ZIndex = 5,
}, scroll)
corner(curLine, 3)

local view = mk("TextLabel", {
	Size = UDim2.new(0, 100, 0, 20),
	Position = UDim2.fromOffset(40, 0),
	BackgroundTransparency = 1,
	RichText = true,
	Text = "",
	TextColor3 = T.text,
	TextSize = SIZE,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextYAlignment = Enum.TextYAlignment.Top,
	TextWrapped = false,
	ZIndex = 7,
}, scroll)
bind(view, "TextColor3", "text")

local gutter = mk("TextLabel", {
	Size = UDim2.new(0, 30, 0, 20),
	BackgroundTransparency = 1,
	RichText = true,
	Text = "1",
	TextColor3 = T.faint,
	TextSize = SIZE,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Right,
	TextYAlignment = Enum.TextYAlignment.Top,
	TextWrapped = false,
	ZIndex = 8,
}, scroll)

local ed = mk("TextBox", {
	Size = UDim2.new(0, 100, 0, 20),
	Position = UDim2.fromOffset(40, 0),
	BackgroundTransparency = 1,
	Text = "",
	TextColor3 = T.text,
	TextSize = SIZE,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextYAlignment = Enum.TextYAlignment.Top,
	TextWrapped = false,
	MultiLine = true,
	ClearTextOnFocus = false,
	ZIndex = 8,
}, scroll)
bind(ed, "TextColor3", "text")

local hit = mk("TextButton", {
	Size = UDim2.new(0, 100, 0, 20),
	Position = UDim2.fromOffset(40, 0),
	BackgroundTransparency = 1,
	Text = "",
	AutoButtonColor = false,
	ZIndex = 13,
}, scroll)

local arrow = mk("TextLabel", {
	Size = UDim2.fromOffset(8, LH),
	Position = UDim2.fromOffset(31, 0),
	BackgroundTransparency = 1,
	Text = "›",
	TextColor3 = T.accent,
	TextSize = 10,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Center,
	ZIndex = 9,
}, scroll)
bind(arrow, "TextColor3", "accent")

local outLine = hairline(well, 1, -55)

local out = mk("TextLabel", {
	Size = UDim2.new(1, -16, 0, 44),
	Position = UDim2.new(0, 8, 1, -51),
	BackgroundTransparency = 1,
	RichText = true,
	Text = "",
	TextColor3 = T.dim,
	TextSize = 10,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextYAlignment = Enum.TextYAlignment.Top,
	TextWrapped = false,
	TextTruncate = Enum.TextTruncate.AtEnd,
	ZIndex = 6,
}, well)
bind(out, "TextColor3", "dim")

-- status ----------------------------------------------------------------------
local status = mk("Frame", {
	Size = UDim2.new(1, 0, 0, STAT_H),
	Position = UDim2.new(0, 0, 1, -STAT_H - 2),
	BackgroundTransparency = 1,
	ZIndex = 4,
}, win)
hairline(status, 0, 0)

local stLeft = mk("TextLabel", {
	Size = UDim2.new(0.56, -12, 1, 0),
	Position = UDim2.fromOffset(12, 0),
	BackgroundTransparency = 1,
	RichText = true,
	Text = "",
	TextColor3 = T.faint,
	TextSize = 10,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Left,
	TextTruncate = Enum.TextTruncate.AtEnd,
	ZIndex = 5,
}, status)

local stRight = mk("TextLabel", {
	Size = UDim2.new(0.44, -24, 1, 0),
	AnchorPoint = Vector2.new(1, 0),
	Position = UDim2.new(1, -22, 0, 0),
	BackgroundTransparency = 1,
	Text = "",
	TextColor3 = T.faint,
	TextSize = 10,
	Font = FONT,
	TextXAlignment = Enum.TextXAlignment.Right,
	ZIndex = 5,
}, status)
bind(stRight, "TextColor3", "faint")

local grip = mk("TextButton", {
	Size = UDim2.fromOffset(16, 16),
	AnchorPoint = Vector2.new(1, 1),
	Position = UDim2.new(1, -5, 1, -3),
	BackgroundTransparency = 1,
	Text = "",
	AutoButtonColor = false,
	ZIndex = 12,
}, win)

for i = 1, 3 do
	local l = mk("Frame", {
		Size = UDim2.fromOffset(8 - i, 1),
		AnchorPoint = Vector2.new(1, 1),
		Position = UDim2.new(1, 0, 1, -i * 4),
		BackgroundColor3 = T.faint,
		BackgroundTransparency = 0.3,
		BorderSizePixel = 0,
		ZIndex = 12,
	}, grip)
	corner(l, 999)
end

-- syntax highlight ------------------------------------------------------------
local KW = {}
for w in ("local function end if then else elseif for while do return and or not nil true false in repeat until break continue export type"):gmatch("%S+") do
	KW[w] = true
end

local BI = {}
for w in ("game workspace script print warn task wait spawn delay pcall xpcall loadstring require getgenv setclipboard pairs ipairs next select type typeof tostring tonumber error math table string os coroutine Instance Enum Vector2 Vector3 CFrame Color3 UDim2 UDim TextService TweenInfo Font"):gmatch("%S+") do
	BI[w] = true
end

local function esc(s)
	return (tostring(s):gsub("&", "&amp;"):gsub("<", "&lt;"):gsub(">", "&gt;"))
end

local function paint(color, text)
	return ('<font color="%s">%s</font>'):format(color, esc(text))
end

local function hl(src)
	if src == "" then return "" end
	local buf = {}
	local i, n = 1, #src
	while i <= n do
		local c = src:sub(i, i)
		local d = src:sub(i + 1, i + 1)

		if c == "-" and d == "-" then
			local j = src:find("\n", i, true) or (n + 1)
			buf[#buf + 1] = paint(T.com, src:sub(i, j - 1))
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
			buf[#buf + 1] = paint(T.str, src:sub(i, j - 1))
			i = j
		elseif c:match("[%a_]") then
			local j = i
			while j <= n and src:sub(j, j):match("[%w_]") do j = j + 1 end
			local word = src:sub(i, j - 1)
			if KW[word] then
				buf[#buf + 1] = paint(T.kw, word)
			elseif BI[word] then
				buf[#buf + 1] = paint(T.bi, word)
			else
				local k = j
				while k <= n and src:sub(k, k):match("[ \t]") do k = k + 1 end
				buf[#buf + 1] = paint(src:sub(k, k) == "(" and T.fnc or T.base, word)
			end
			i = j
		elseif c:match("%d") then
			local j = i
			while j <= n and src:sub(j, j):match("[%w_%.]") do j = j + 1 end
			buf[#buf + 1] = paint(T.num, src:sub(i, j - 1))
			i = j
		else
			local j = i
			while j <= n do
				local ch = src:sub(j, j)
				if ch:match("[%a_]") or ch:match("%d") or ch == "-" or ch == '"' or ch == "'" then break end
				j = j + 1
			end
			if j == i then j = i + 1 end
			buf[#buf + 1] = paint(T.punc, src:sub(i, j - 1))
			i = j
		end
	end
	return table.concat(buf)
end

-- estado ----------------------------------------------------------------------
local logs = {}
local stateLabel, stateKind = "pronto", "dim"
local lastMs = nil
local errorLine = nil
local currentLine = 1
local cursorLine

pcall(function()
	cursorLine = function()
		local ok, pos = pcall(function() return ed.CursorPosition end)
		if not ok or type(pos) ~= "number" or pos < 0 then return nil end
		return select(2, ed.Text:sub(1, pos):gsub("\n", "")) + 1
	end
end)

local function kindColor(kind)
	if kind == "ok" then return hex(T.accent) end
	if kind == "bad" then return hex(T.red) end
	if kind == "warn" then return hex(T.num) end
	return hex(T.faint)
end

local function setState(label, kind)
	stateLabel, stateKind = label, kind
	local extra = lastMs and ('   <font color="%s">· %d ms</font>'):format(hex(T.faint), lastMs) or ""
	stLeft.Text = ('<font color="%s">luau</font>  <font color="%s">%s</font>%s'):format(hex(T.accent), kindColor(kind), label, extra)
end

local function stamp()
	local ok, s = pcall(function() return os.date("%H:%M:%S") end)
	return ok and s or ""
end

local function log(msg, kind)
	local color = kind == "bad" and hex(T.red) or kind == "ok" and hex(T.accent) or hex(T.faint)
	local text = esc(msg):gsub("[\r\n\t]+", " ")
	if #text > 96 then text = text:sub(1, 93) .. "..." end
	logs[#logs + 1] = ('<font color="#3A404B">%s</font>  <font color="%s">%s</font>'):format(stamp(), color, text)
	while #logs > 3 do table.remove(logs, 1) end
	out.Text = table.concat(logs, "\n")
end

local function paintGutter()
	local text = ed.Text
	local lines = text == "" and 1 or select(2, text:gsub("\n", "")) + 1
	local parts = {}
	for i = 1, lines do
		local color
		if i == errorLine then
			color = hex(T.red)
		elseif i == currentLine then
			color = hex(T.accent)
		else
			color = hex(T.faint)
		end
		parts[i] = ('<font color="%s">%d</font>'):format(color, i)
	end
	gutter.Text = table.concat(parts, "\n")
end

local function syncBands()
	curLine.Position = UDim2.fromOffset(0, (currentLine - 1) * LH)
	curLine.Size = UDim2.new(1, 0, 0, LH)
	arrow.Position = UDim2.fromOffset(31, (currentLine - 1) * LH)
	arrow.Size = UDim2.fromOffset(8, LH)
	if errorLine then
		errLine.Visible = true
		errLine.Position = UDim2.fromOffset(0, (errorLine - 1) * LH)
		errLine.Size = UDim2.new(1, 0, 0, LH)
	else
		errLine.Visible = false
	end
	paintGutter()
end

local function updateCursor()
	if cursorLine then
		local l = cursorLine()
		if l then currentLine = l end
	end
	syncBands()
end

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

	empty.Visible = text == "" and not ed:IsFocused()
	emptyHint.Visible = empty.Visible

	syncBands()

	local size = #text >= 1024 and ("%.1f KB"):format(#text / 1024) or ("%d B"):format(#text)
	stRight.Text = ("%d ln · %s"):format(lines, size)
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

local function startShimmer()
	shimmer.Visible = true
	if shimmerTween then shimmerTween:Cancel() end
	shimmer.Position = UDim2.new(-0.4, 0, 1, -1)
	shimmerTween = Tween:Create(shimmer, TweenInfo.new(1.1, Enum.EasingStyle.Linear, Enum.EasingDirection.Out, -1), {
		Position = UDim2.new(1.4, 0, 1, -1),
	})
	shimmerTween:Play()
end

local function stopShimmer()
	if shimmerTween then shimmerTween:Cancel() end
	shimmer.Visible = false
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
		errorLine = tonumber(tostring(err):match("console:(%d+)"))
		setState("erro de sintaxe", "bad")
		log(tostring(err), "bad")
		syncBands()
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

	lastMs = nil
	errorLine = nil
	syncBands()
	setState("executando", "warn")
	startShimmer()
	tween(winStroke, { Color = T.accent, Transparency = 0.05 }, 0.1)
	task.delay(0.22, function()
		tween(winStroke, { Color = T.line, Transparency = 0.2 }, 0.3)
	end)
	log("executando script", "dim")

	local t0 = os.clock()
	task.spawn(function()
		local ok, res = pcall(fn)
		lastMs = math.floor((os.clock() - t0) * 1000)
		stopShimmer()
		if ok then
			setState("ok", "ok")
			if res ~= nil then log("retorno: " .. tostring(res), "ok") end
		else
			local msg = tostring(res)
			errorLine = tonumber(msg:match("console:(%d+)"))
			setState("erro", "bad")
			log(msg, "bad")
			syncBands()
			if errorLine then
				scroll.CanvasPosition = Vector2.new(0, math.max(0, (errorLine - 1) * LH - 30))
			end
		end
	end)
end

-- tema e fonte ----------------------------------------------------------------
local function applyTheme(th)
	T = th
	for _, b in ipairs(binds) do
		b[1][b[2]] = T[b[3]]
	end
	shimmerGrad.Color = ColorSequence.new(T.accent, T.accent)
	themeBtn.Text = T.name:sub(1, 5)
	render()
	layout()
	setState(stateLabel, stateKind)
end

local function zoom(delta)
	local n = math.clamp(SIZE + delta, 10, 16)
	if n == SIZE then return end
	SIZE = n
	LH = math.max(n + 2, math.floor(TextService:GetTextSize("A\nA", n, FONT, Vector2.new(4000, 4000)).Y / 2 + 0.5))
	ed.TextSize, view.TextSize, gutter.TextSize, arrow.TextSize = n, n, n, n
	scroll.ScrollBarThickness = 3
	layout()
	render()
end

-- acoes -----------------------------------------------------------------------
local function clearOutput()
	logs = {}
	out.Text = ""
end

runBtn.MouseButton1Click:Connect(run)

clearBtn.MouseButton1Click:Connect(function()
	ed.Text = ""
	clearOutput()
	errorLine = nil
	lastMs = nil
	setState("pronto", "dim")
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

smallBtn.MouseButton1Click:Connect(function() zoom(-1) end)
bigBtn.MouseButton1Click:Connect(function() zoom(1) end)

local themeIndex = 1
themeBtn.MouseButton1Click:Connect(function()
	themeIndex = themeIndex % #THEMES + 1
	applyTheme(THEMES[themeIndex])
end)

ed:GetPropertyChangedSignal("Text"):Connect(function()
	layout()
	if ed:IsFocused() then follow() end
end)

pcall(function()
	ed:GetPropertyChangedSignal("CursorPosition"):Connect(updateCursor)
end)

ed.Focused:Connect(function()
	ed.TextTransparency = 0
	ed.ZIndex = 12
	view.ZIndex = 7
	hit.Visible = false
	layout()
	updateCursor()
	setState("editando", "ok")
end)

ed.FocusLost:Connect(function(enter)
	ed.TextTransparency = 1
	ed.ZIndex = 8
	render()
	view.ZIndex = 11
	hit.Visible = true
	layout()
	setState("pronto", "dim")
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

-- arrastar, inercia e snap ----------------------------------------------------
local dragging, dragStart, dragPos = false, nil, nil
local samples = {}

local function snapEdges()
	local x, y = win.Position.X.Offset, win.Position.Y.Offset
	local sw, sh = win.AbsoluteSize.X, win.AbsoluteSize.Y
	local nx, ny = x, y
	if x < 12 then nx = 8 elseif x + sw > vp.X - 12 then nx = vp.X - sw - 8 end
	if y < 12 then ny = 8 elseif y + sh > vp.Y - 12 then ny = vp.Y - sh - 8 end
	if nx ~= x or ny ~= y then
		tween(win, { Position = UDim2.fromOffset(nx, ny) }, 0.16, Enum.EasingStyle.Quint)
	end
end

bar.InputBegan:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
		dragging = true
		dragStart = i.Position
		dragPos = win.Position
		samples = {}
		local c
		c = i.Changed:Connect(function()
			if i.UserInputState == Enum.UserInputState.End and dragging then
				dragging = false
				c:Disconnect()
				local a, b = samples[#samples - 1], samples[#samples]
				if a and b and b.t - a.t > 0 and b.t - a.t < 0.12 then
					local v = (b.p - a.p) / (b.t - a.t)
					local d = v * 0.05
					local target = UDim2.fromOffset(
						math.clamp(win.Position.X.Offset + d.X, -60, vp.X),
						math.clamp(win.Position.Y.Offset + d.Y, -60, vp.Y)
					)
					tween(win, { Position = target }, 0.18, Enum.EasingStyle.Quint).Completed:Connect(snapEdges)
				else
					snapEdges()
				end
			end
		end)
	end
end)

on(UIS.InputChanged, function(i)
	if not dragging then return end
	if i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch then
		local d = i.Position - dragStart
		win.Position = UDim2.fromOffset(dragPos.X.Offset + d.X, dragPos.Y.Offset + d.Y)
		samples[#samples + 1] = { p = i.Position, t = os.clock() }
		if #samples > 6 then table.remove(samples, 1) end
		syncShadow()
	end
end)

-- redimensionar ---------------------------------------------------------------
local sizing, sizeStart, size0 = false, nil, nil

grip.InputBegan:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
		sizing = true
		sizeStart = i.Position
		size0 = win.AbsoluteSize
		local c
		c = i.Changed:Connect(function()
			if i.UserInputState == Enum.UserInputState.End and sizing then
				sizing = false
				c:Disconnect()
				snapEdges()
			end
		end)
	end
end)

on(UIS.InputChanged, function(i)
	if not sizing then return end
	if i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch then
		local d = i.Position - sizeStart
		win.Size = UDim2.fromOffset(
			math.clamp(size0.X + d.X, 300, vp.X - 10),
			math.clamp(size0.Y + d.Y, 330, vp.Y - 10)
		)
		layout()
		syncShadow()
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
	task.delay(0.18, function()
		layout()
		syncShadow()
	end)
end

local function minimize()
	minimized = not minimized
	if minimized then
		miniSize, miniPos = win.Size, win.Position
		tabs.Visible = false
		tool.Visible = false
		well.Visible = false
		status.Visible = false
		win.ClipsDescendants = true
		tween(win, { Size = UDim2.fromOffset(win.AbsoluteSize.X, TITLE_H) }, 0.15, Enum.EasingStyle.Quint)
	else
		tabs.Visible = true
		tool.Visible = true
		well.Visible = true
		status.Visible = true
		win.Position = miniPos
		tween(win, { Size = miniSize }, 0.16, Enum.EasingStyle.Quint)
		task.delay(0.16, function()
			if not minimized then win.ClipsDescendants = false end
			layout()
			syncShadow()
		end)
	end
	syncShadow()
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

-- janela inativa --------------------------------------------------------------
local dim = mk("Frame", {
	Size = UDim2.new(1, 0, 1, 0),
	BackgroundColor3 = Color3.new(0, 0, 0),
	BackgroundTransparency = 1,
	BorderSizePixel = 0,
	Active = false,
	ZIndex = 20,
}, win)
corner(dim, 12)

local function setActive(state)
	tween(dim, { BackgroundTransparency = state and 1 or 0.62 }, 0.16)
	tween(title, { TextColor3 = state and T.dim or T.faint }, 0.16)
end

win.InputBegan:Connect(function(i)
	if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
		setActive(true)
	end
end)

on(UIS.InputBegan, function(i, gp)
	if gp then return end
	if i.UserInputType ~= Enum.UserInputType.MouseButton1 and i.UserInputType ~= Enum.UserInputType.Touch then return end
	local p, s = win.AbsolutePosition, win.AbsoluteSize
	local ip = i.Position
	if ip.X < p.X or ip.Y < p.Y or ip.X > p.X + s.X or ip.Y > p.Y + s.Y then
		setActive(false)
	end
end)

-- teclado do celular ----------------------------------------------------------
if isTouch then
	local raised, home = false, win.Position
	ed.Focused:Connect(function()
		if raised then return end
		raised = true
		home = win.Position
		tween(win, { Position = UDim2.fromOffset(home.X.Offset, 10) }, 0.16).Completed:Connect(syncShadow)
	end)
	ed.FocusLost:Connect(function()
		if not raised then return end
		raised = false
		tween(win, { Position = home }, 0.16).Completed:Connect(syncShadow)
	end)
end

pcall(function()
	getgenv().TerminalLuau = { close = close }
end)

-- abrir -----------------------------------------------------------------------
local openScale = Instance.new("UIScale")
openScale.Scale = 0.97
openScale.Parent = win
tween(openScale, { Scale = 1 }, 0.14, Enum.EasingStyle.Quint)

LH = math.max(SIZE + 2, math.floor(TextService:GetTextSize("A\nA", SIZE, FONT, Vector2.new(4000, 4000)).Y / 2 + 0.5))
ed.TextTransparency = 1
view.ZIndex = 11
hit.Visible = true
setState("pronto", "dim")
layout()
render()
syncShadow()
task.defer(function()
	layout()
	syncShadow()
end)
