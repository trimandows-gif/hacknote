--============================================================
-- HAK NOTE
-- CONTROL CENTER
--
-- FEATURES
-- • Hak Note logo
-- • ` key opens/closes menu
-- • Collapsible menu
-- • Flight + configurable key/speed
-- • Smooth NPC lock-on
-- • NPC hover positioning
-- • 20-stud nearby scanner
-- • All nearby object names + distances within 20 studs\n-- • Search object names and apply ESP
-- • Whole-map fruit scanner every 20 seconds
-- • Instant fruit detection
-- • Fruit world markers
-- • Player coordinates
-- • Friend coordinate markers
-- • Custom marker colors
--
-- NO AUTO ATTACK
-- NO LAG DEFUSER
-- NO INVULNERABILITY
--============================================================

local Players = game:GetService("Players")
local UserInputService = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local Workspace = game:GetService("Workspace")
local TweenService = game:GetService("TweenService")
local VirtualInputManager = game:GetService("VirtualInputManager")

local Player = Players.LocalPlayer
local PlayerGui = Player:WaitForChild("PlayerGui")
local IsMobile = UserInputService.TouchEnabled

--============================================================
-- CONFIG
--============================================================

local LOGO_ID = "rbxassetid://109914112670196"

local MENU_KEY = Enum.KeyCode.Backquote
local FLIGHT_KEY = Enum.KeyCode.F

local FLIGHT_SPEED = 70

local NPC_RANGE = 250
local NPC_SCAN_RATE = 0.15

local NPC_HOVER_HEIGHT = 12
local NPC_HOVER_DISTANCE = 5
local NPC_MOVE_SMOOTHNESS = 0.10

local NEARBY_RANGE = 20
local NEARBY_SCAN_RATE = 0.15

local FRUIT_SCAN_INTERVAL = 20
local FRUIT_SCAN_BATCH = 200
local MAX_FRUIT_MARKERS = 100

local MARKER_UPDATE_RATE = 0.25

--============================================================
-- FRUITS
--============================================================

local FRUIT_NAMES = {
	"Rocket Fruit",
	"Spin Fruit",
	"Spring Fruit",
	"Spike Fruit",
	"Bomb Fruit",
	"Smoke Fruit",
	"Blade Fruit",
	"Sand Fruit",
	"Barrier Fruit",
	"Diamond Fruit",
	"Rubber Fruit",
	"Ghost Fruit",
	"Love Fruit",
	"Creation Fruit",
	"Falcon Fruit",
	"Flame Fruit",
	"Ice Fruit",
	"Dark Fruit",
	"Quake Fruit",
	"Spider Fruit",
	"Sound Fruit",
	"Gravity Fruit",
	"Light Fruit",
	"Magma Fruit",
	"Phoenix Fruit",
	"Pain Fruit",
	"Blizzard Fruit",
	"Shadow Fruit",
	"Portal Fruit",
	"Venom Fruit",
	"Control Fruit",
	"Spirit Fruit",
	"Mammoth Fruit",
	"T-Rex Fruit",
	"Yeti Fruit",
	"Buddha Fruit",
	"Lightning Fruit",
	"Tiger Fruit",
	"Gas Fruit",
	"Dough Fruit",
	"Kitsune Fruit",
	"Dragon Fruit",
}

local FruitLookup = {}

for _, fruit in ipairs(FRUIT_NAMES) do
	FruitLookup[string.lower(fruit)] = fruit
end

--============================================================
-- STATE
--============================================================

local Character
local Humanoid
local RootPart

local MenuOpen = true
local MenuCollapsed = false

local FlightEnabled = false
local WaitingForFlightKey = false
local MobileAscend = false
local MobileDescend = false

local NPCEnabled = false
local CurrentNPC = nil

local NearbyObjects = {}
local RadarEnabled = true
local ESPHighlights = {}
local ClearESP

local FruitMarkers = {}
local KnownFruits = {}

local FriendMarkers = {}

local FruitScanRunning = false
local FruitScanRequested = false

local FruitScanTimer = FRUIT_SCAN_INTERVAL
local NearbyTimer = 0
local NPCScanTimer = 0
local MarkerTimer = 0

local FlightVelocity
local FlightOrientation
local FlightAttachment

--============================================================
-- CHARACTER
--============================================================

local function UpdateCharacter(character)

	Character = character

	Humanoid =
		character:WaitForChild(
			"Humanoid",
			10
		)

	RootPart =
		character:WaitForChild(
			"HumanoidRootPart",
			10
		)

end

if Player.Character then
	UpdateCharacter(Player.Character)
end

Player.CharacterAdded:Connect(function(character)

	UpdateCharacter(character)

	if FlightEnabled then
		task.wait(0.25)

		if RootPart and Humanoid then
			-- Flight is recreated below.
		end
	end

end)

--============================================================
-- OLD GUI CLEANUP
--============================================================

local OldGui =
	PlayerGui:FindFirstChild("HakNote")

if OldGui then
	OldGui:Destroy()
end

--============================================================
-- GUI
--============================================================

local Gui =
	Instance.new("ScreenGui")

Gui.Name = "HakNote"
Gui.ResetOnSpawn = false
Gui.IgnoreGuiInset = true
Gui.DisplayOrder = 999
Gui.Parent = PlayerGui

--============================================================
-- MAIN WINDOW
--============================================================

local Main =
	Instance.new("Frame")

Main.Size =
	UDim2.fromOffset(
		390,
		640
	)

Main.Position =
	UDim2.new(
		0,
		25,
		0.15,
		0
	)

Main.BackgroundColor3 =
	Color3.fromRGB(
		12,
		14,
		19
	)

Main.BorderSizePixel = 0
Main.Active = true
Main.Parent = Gui

local FlightTouchControls = Instance.new("Frame")
FlightTouchControls.Name = "MobileFlightControls"
FlightTouchControls.Size = UDim2.fromOffset(116, 48)
FlightTouchControls.Position = UDim2.new(1, -128, 1, -128)
FlightTouchControls.BackgroundColor3 = Color3.fromRGB(12, 14, 19)
FlightTouchControls.BackgroundTransparency = 0.12
FlightTouchControls.BorderSizePixel = 0
FlightTouchControls.Visible = false
FlightTouchControls.Parent = Gui
local FlightTouchCorner = Instance.new("UICorner")
FlightTouchCorner.CornerRadius = UDim.new(0, 9)
FlightTouchCorner.Parent = FlightTouchControls

local function MakeFlightTouchButton(text, x, setState)
	local button = Instance.new("TextButton")
	button.Size = UDim2.fromOffset(48, 38)
	button.Position = UDim2.fromOffset(x, 5)
	button.BackgroundColor3 = Color3.fromRGB(35, 90, 120)
	button.BorderSizePixel = 0
	button.Text = text
	button.TextColor3 = Color3.new(1, 1, 1)
	button.Font = Enum.Font.GothamBlack
	button.TextSize = 14
	button.Parent = FlightTouchControls
	local corner = Instance.new("UICorner")
	corner.CornerRadius = UDim.new(0, 7)
	corner.Parent = button
	button.InputBegan:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.Touch
			or input.UserInputType == Enum.UserInputType.MouseButton1 then
			setState(true)
		end
	end)
	button.InputEnded:Connect(function(input)
		if input.UserInputType == Enum.UserInputType.Touch
			or input.UserInputType == Enum.UserInputType.MouseButton1 then
			setState(false)
		end
	end)
	return button
end

MakeFlightTouchButton("UP", 5, function(value) MobileAscend = value end)
MakeFlightTouchButton("DOWN", 63, function(value) MobileDescend = value end)

local MainScale = Instance.new("UIScale")
MainScale.Scale = IsMobile and 0.78 or 1
MainScale.Parent = Main
if IsMobile then
	Main.Position = UDim2.new(0.5, -195, 0.5, -320)
end

local MainCorner =
	Instance.new("UICorner")

MainCorner.CornerRadius =
	UDim.new(0, 14)

MainCorner.Parent = Main

local MainStroke =
	Instance.new("UIStroke")

MainStroke.Color =
	Color3.fromRGB(
		65,
		70,
		85
	)

MainStroke.Thickness = 1
MainStroke.Parent = Main

--============================================================
-- HEADER
--============================================================

local Header =
	Instance.new("Frame")

Header.Size =
	UDim2.new(
		1,
		0,
		0,
		72
	)

Header.BackgroundColor3 =
	Color3.fromRGB(
		19,
		21,
		27
	)

Header.BorderSizePixel = 0
Header.Active = true
Header.Parent = Main

local HeaderCorner =
	Instance.new("UICorner")

HeaderCorner.CornerRadius =
	UDim.new(0, 14)

HeaderCorner.Parent = Header

local Logo =
	Instance.new("ImageLabel")

Logo.Size =
	UDim2.fromOffset(
		56,
		56
	)

Logo.Position =
	UDim2.fromOffset(
		9,
		8
	)

Logo.BackgroundTransparency = 1
Logo.Image = LOGO_ID
Logo.ScaleType =
	Enum.ScaleType.Fit

Logo.Parent = Header

local Title =
	Instance.new("TextLabel")

Title.Size =
	UDim2.new(
		1,
		-150,
		0,
		28
	)

Title.Position =
	UDim2.fromOffset(
		75,
		8
	)

Title.BackgroundTransparency = 1
Title.Text = "HAK NOTE"

Title.TextColor3 =
	Color3.fromRGB(
		245,
		245,
		250
	)

Title.Font =
	Enum.Font.GothamBlack

Title.TextSize = 20
Title.TextXAlignment =
	Enum.TextXAlignment.Left

Title.Parent = Header

local Subtitle =
	Instance.new("TextLabel")

Subtitle.Size =
	UDim2.new(
		1,
		-150,
		0,
		20
	)

Subtitle.Position =
	UDim2.fromOffset(
		76,
		37
	)

Subtitle.BackgroundTransparency = 1
Subtitle.Text =
	"CONTROL CENTER"

Subtitle.TextColor3 =
	Color3.fromRGB(
		125,
		130,
		145
	)

Subtitle.Font =
	Enum.Font.GothamBold

Subtitle.TextSize = 10
Subtitle.TextXAlignment =
	Enum.TextXAlignment.Left

Subtitle.Parent = Header

--============================================================
-- MINIMIZE
--============================================================

local Minimize =
	Instance.new("TextButton")

Minimize.Size =
	UDim2.fromOffset(
		30,
		30
	)

Minimize.Position =
	UDim2.new(
		1,
		-72,
		0,
		21
	)

Minimize.BackgroundColor3 =
	Color3.fromRGB(
		40,
		43,
		52
	)

Minimize.BorderSizePixel = 0
Minimize.Text = "−"

Minimize.TextColor3 =
	Color3.new(
		1,
		1,
		1
	)

Minimize.Font =
	Enum.Font.GothamBold

Minimize.TextSize = 18
Minimize.Parent = Header

local MinCorner =
	Instance.new("UICorner")

MinCorner.CornerRadius =
	UDim.new(0, 7)

MinCorner.Parent = Minimize

--============================================================
-- CLOSE
--============================================================

local Close =
	Instance.new("TextButton")

Close.Size =
	UDim2.fromOffset(
		30,
		30
	)

Close.Position =
	UDim2.new(
		1,
		-37,
		0,
		21
	)

Close.BackgroundColor3 =
	Color3.fromRGB(
		105,
		42,
		48
	)

Close.BorderSizePixel = 0
Close.Text = "×"

Close.TextColor3 =
	Color3.new(
		1,
		1,
		1
	)

Close.Font =
	Enum.Font.GothamBold

Close.TextSize = 18
Close.Parent = Header

local CloseCorner =
	Instance.new("UICorner")

CloseCorner.CornerRadius =
	UDim.new(0, 7)

CloseCorner.Parent = Close

local ReopenButton = Instance.new("TextButton")
ReopenButton.Name = "HakNoteReopen"
ReopenButton.Size = UDim2.fromOffset(52, 42)
ReopenButton.Position = UDim2.new(0, 12, 0.5, -21)
ReopenButton.BackgroundColor3 = Color3.fromRGB(20, 85, 120)
ReopenButton.BorderSizePixel = 0
ReopenButton.Text = "HAK"
ReopenButton.TextColor3 = Color3.new(1, 1, 1)
ReopenButton.Font = Enum.Font.GothamBlack
ReopenButton.TextSize = 13
ReopenButton.Visible = false
ReopenButton.Parent = Gui
local ReopenCorner = Instance.new("UICorner")
ReopenCorner.CornerRadius = UDim.new(0, 9)
ReopenCorner.Parent = ReopenButton

ReopenButton.Activated:Connect(function()
	MenuOpen = true
	Main.Visible = true
	ReopenButton.Visible = false
end)

--============================================================
-- CONTENT
--============================================================

local Content =
	Instance.new("ScrollingFrame")

Content.Size =
	UDim2.new(
		1,
		-10,
		1,
		-82
	)

Content.Position =
	UDim2.fromOffset(
		5,
		77
	)

Content.BackgroundTransparency = 1
Content.BorderSizePixel = 0

Content.ScrollBarThickness = 4
Content.ScrollBarImageTransparency = 0.4

Content.CanvasSize =
	UDim2.fromOffset(
		0,
		1060
	)

Content.Parent = Main

local Padding =
	Instance.new("UIPadding")

Padding.PaddingLeft =
	UDim.new(0, 7)

Padding.PaddingRight =
	UDim.new(0, 7)

Padding.Parent = Content

--============================================================
-- SECTION
--============================================================

local function Section(text, y)

	local label =
		Instance.new("TextLabel")

	label.Size =
		UDim2.new(
			1,
			-14,
			0,
			24
		)

	label.Position =
		UDim2.fromOffset(
			7,
			y
		)

	label.BackgroundTransparency = 1

	label.Text =
		"  " .. text

	label.TextColor3 =
		Color3.fromRGB(
			115,
			190,
			255
		)

	label.Font =
		Enum.Font.GothamBlack

	label.TextSize = 11

	label.TextXAlignment =
		Enum.TextXAlignment.Left

	label.Parent = Content

	return label
end

--============================================================
-- BUTTON
--============================================================

local function Button(
	text,
	y,
	callback
)

	local button =
		Instance.new("TextButton")

	button.Size =
		UDim2.new(
			1,
			-14,
			0,
			36
		)

	button.Position =
		UDim2.fromOffset(
			7,
			y
		)

	button.BackgroundColor3 =
		Color3.fromRGB(
			31,
			34,
			42
		)

	button.BorderSizePixel = 0
	button.Text = text

	button.TextColor3 =
		Color3.fromRGB(
			235,
			237,
			242
		)

	button.Font =
		Enum.Font.GothamBold

	button.TextSize = 11

	button.Parent = Content

	local corner =
		Instance.new("UICorner")

	corner.CornerRadius =
		UDim.new(0, 7)

	corner.Parent = button

	button.MouseButton1Click:Connect(
		callback
	)

	return button
end

--============================================================
-- INPUT BOX
--============================================================

local function InputBox(
	placeholder,
	value,
	y
)

	local box =
		Instance.new("TextBox")

	box.Size =
		UDim2.new(
			1,
			-14,
			0,
			30
		)

	box.Position =
		UDim2.fromOffset(
			7,
			y
		)

	box.BackgroundColor3 =
		Color3.fromRGB(
			24,
			27,
			34
		)

	box.BorderSizePixel = 0

	box.PlaceholderText =
		placeholder

	box.PlaceholderColor3 =
		Color3.fromRGB(
			100,
			105,
			118
		)

	box.Text =
		value or ""

	box.TextColor3 =
		Color3.new(
			1,
			1,
			1
		)

	box.Font =
		Enum.Font.Gotham

	box.TextSize = 11

	box.ClearTextOnFocus = false

	box.Parent = Content

	local corner =
		Instance.new("UICorner")

	corner.CornerRadius =
		UDim.new(0, 6)

	corner.Parent = box

	return box
end

--============================================================
-- STATUS LABEL
--============================================================

local function Status(
	text,
	y
)

	local label =
		Instance.new("TextLabel")

	label.Size =
		UDim2.new(
			1,
			-14,
			0,
			25
		)

	label.Position =
		UDim2.fromOffset(
			7,
			y
		)

	label.BackgroundTransparency = 1

	label.Text = text

	label.TextColor3 =
		Color3.fromRGB(
			155,
			160,
			175
		)

	label.Font =
		Enum.Font.Gotham

	label.TextSize = 10

	label.TextXAlignment =
		Enum.TextXAlignment.Left

	label.Parent = Content

	return label
end

--============================================================
-- NPC SECTION
--============================================================

Section(
	"NPC LOCK-ON",
	0
)

local NPCButton =
	Button(
		"NPC LOCK-ON  •  OFF",
	28,
		function()

			NPCEnabled =
				not NPCEnabled

			if not NPCEnabled then
				CurrentNPC = nil
			end
		end
	)

local NPCRangeBox =
	InputBox(
		"NPC range",
		tostring(NPC_RANGE),
		70
	)

NPCRangeBox.FocusLost:Connect(
	function()

		local value =
			tonumber(
				NPCRangeBox.Text
			)

		if value then

			NPC_RANGE =
				math.clamp(
					value,
					10,
					100000
				)

			NPCRangeBox.Text =
				tostring(
					NPC_RANGE
				)
		end
	end
)

local HoverHeightBox =
	InputBox(
		"Hover height",
		tostring(NPC_HOVER_HEIGHT),
		105
)

HoverHeightBox.FocusLost:Connect(
	function()

		local value =
			tonumber(
				HoverHeightBox.Text
			)

		if value then
			NPC_HOVER_HEIGHT =
				math.clamp(
					value,
					1,
					100
				)
		end
	end
)

local NPCStatus =
	Status(
		"NPC: NONE",
		140
	)

--============================================================
-- NEARBY RADAR
--============================================================

Section(
	"20 STUD RADAR",
	175
)

local RadarButton =
	Button(
		"20 STUD RADAR  •  ON",
	203,
		function()
			RadarEnabled = not RadarEnabled
			RadarPanel.Visible = RadarEnabled
			RadarButton.Text = RadarEnabled and "20 STUD RADAR  •  ON" or "20 STUD RADAR  •  OFF"
			if not RadarEnabled then
				ClearESP()
				RadarStatus.Text = "Radar and ESP detection: OFF"
			end
		end
	)

local RadarStatus =
	Status(
		"Nearby objects: 0",
		245
	)

-- Side panel: nearby object list and name-based ESP.
local RadarPanel = Instance.new("Frame")
RadarPanel.Name = "HakNoteRadarPanel"
RadarPanel.Size = UDim2.fromOffset(300, 330)
RadarPanel.Position = UDim2.new(1, -320, 0, 100)
RadarPanel.BackgroundColor3 = Color3.fromRGB(12, 14, 19)
RadarPanel.BorderSizePixel = 0
RadarPanel.Visible = true
RadarPanel.Parent = Gui

local RadarPanelScale = Instance.new("UIScale")
RadarPanelScale.Scale = IsMobile and 0.72 or 1
RadarPanelScale.Parent = RadarPanel
if IsMobile then
	RadarPanel.Position = UDim2.new(1, -235, 0, 78)
end

local RadarPanelCorner = Instance.new("UICorner")
RadarPanelCorner.CornerRadius = UDim.new(0, 10)
RadarPanelCorner.Parent = RadarPanel

local RadarPanelStroke = Instance.new("UIStroke")
RadarPanelStroke.Color = Color3.fromRGB(65, 70, 85)
RadarPanelStroke.Parent = RadarPanel

local RadarPanelTitle = Instance.new("TextLabel")
RadarPanelTitle.Size = UDim2.new(1, -42, 0, 28)
RadarPanelTitle.Position = UDim2.fromOffset(8, 5)
RadarPanelTitle.BackgroundTransparency = 1
RadarPanelTitle.Text = "20-STUD RADAR"
RadarPanelTitle.TextColor3 = Color3.fromRGB(115, 190, 255)
RadarPanelTitle.Font = Enum.Font.GothamBlack
RadarPanelTitle.TextSize = 13
RadarPanelTitle.TextXAlignment = Enum.TextXAlignment.Left
RadarPanelTitle.Parent = RadarPanel

local RadarCloseButton = Instance.new("TextButton")
RadarCloseButton.Name = "CloseESP"
RadarCloseButton.Size = UDim2.fromOffset(24, 24)
RadarCloseButton.Position = UDim2.new(1, -29, 0, 6)
RadarCloseButton.BackgroundColor3 = Color3.fromRGB(105, 42, 48)
RadarCloseButton.BorderSizePixel = 0
RadarCloseButton.Text = "X"
RadarCloseButton.TextColor3 = Color3.new(1, 1, 1)
RadarCloseButton.Font = Enum.Font.GothamBold
RadarCloseButton.TextSize = 12
RadarCloseButton.Parent = RadarPanel
local RadarCloseCorner = Instance.new("UICorner")
RadarCloseCorner.CornerRadius = UDim.new(0, 6)
RadarCloseCorner.Parent = RadarCloseButton

RadarCloseButton.Activated:Connect(function()
	ClearESP()
	RadarPanel.Visible = false
	RadarEnabled = false
	RadarButton.Text = "20 STUD RADAR  •  OFF"
	RadarStatus.Text = "Radar and ESP detection: OFF"
end)

local RadarList = Instance.new("ScrollingFrame")
RadarList.Size = UDim2.new(1, -16, 0, 205)
RadarList.Position = UDim2.fromOffset(8, 36)
RadarList.BackgroundColor3 = Color3.fromRGB(24, 27, 34)
RadarList.BorderSizePixel = 0
RadarList.ScrollBarThickness = 4
RadarList.Parent = RadarPanel

local RadarListCorner = Instance.new("UICorner")
RadarListCorner.CornerRadius = UDim.new(0, 6)
RadarListCorner.Parent = RadarList

local RadarListText = Instance.new("TextLabel")
RadarListText.Size = UDim2.new(1, -12, 0, 24)
RadarListText.Position = UDim2.fromOffset(6, 5)
RadarListText.AutomaticSize = Enum.AutomaticSize.Y
RadarListText.BackgroundTransparency = 1
RadarListText.Text = "No objects within 20 studs."
RadarListText.TextColor3 = Color3.fromRGB(230, 233, 240)
RadarListText.Font = Enum.Font.Code
RadarListText.TextSize = 12
RadarListText.TextXAlignment = Enum.TextXAlignment.Left
RadarListText.TextYAlignment = Enum.TextYAlignment.Top
RadarListText.Parent = RadarList

local ESPNameBox = Instance.new("TextBox")
ESPNameBox.Size = UDim2.new(1, -16, 0, 30)
ESPNameBox.Position = UDim2.fromOffset(8, 249)
ESPNameBox.BackgroundColor3 = Color3.fromRGB(24, 27, 34)
ESPNameBox.BorderSizePixel = 0
ESPNameBox.PlaceholderText = "Type an object name from the list..."
ESPNameBox.TextColor3 = Color3.new(1, 1, 1)
ESPNameBox.PlaceholderColor3 = Color3.fromRGB(130, 135, 148)
ESPNameBox.Font = Enum.Font.Gotham
ESPNameBox.TextSize = 11
ESPNameBox.ClearTextOnFocus = false
ESPNameBox.Parent = RadarPanel

local ESPNameCorner = Instance.new("UICorner")
ESPNameCorner.CornerRadius = UDim.new(0, 6)
ESPNameCorner.Parent = ESPNameBox

local ESPButton = Instance.new("TextButton")
ESPButton.Size = UDim2.new(1, -16, 0, 32)
ESPButton.Position = UDim2.fromOffset(8, 287)
ESPButton.BackgroundColor3 = Color3.fromRGB(35, 90, 120)
ESPButton.BorderSizePixel = 0
ESPButton.Text = "ESP NAME  •  APPLY"
ESPButton.TextColor3 = Color3.new(1, 1, 1)
ESPButton.Font = Enum.Font.GothamBold
ESPButton.TextSize = 11
ESPButton.Parent = RadarPanel

local ESPButtonCorner = Instance.new("UICorner")
ESPButtonCorner.CornerRadius = UDim.new(0, 6)
ESPButtonCorner.Parent = ESPButton

--============================================================
-- FLIGHT
--============================================================

Section(
	"FLIGHT",
	280
)

local FlightButton =
	Button(
		"FLIGHT  •  OFF",
	308,
		function()

			FlightEnabled =
				not FlightEnabled

			FlightTouchControls.Visible = IsMobile and FlightEnabled

			if FlightEnabled then
				CreateFlight()
			else
				DestroyFlight()
			end
		end
	)

local FlightKeyButton =
	Button(
		"FLIGHT KEY  •  F",
	348,
		function()

			WaitingForFlightKey =
				true

			FlightKeyButton.Text =
				"PRESS A KEY..."
		end
	)

local FlightSpeedBox =
	InputBox(
		"Flight speed",
		tostring(FLIGHT_SPEED),
		388
	)

FlightSpeedBox.FocusLost:Connect(
	function()

		local value =
			tonumber(
				FlightSpeedBox.Text
			)

		if value then

			FLIGHT_SPEED =
				math.clamp(
					value,
					10,
					500
				)

			FlightSpeedBox.Text =
				tostring(
					FLIGHT_SPEED
				)
		end
	end
)

--============================================================
-- FRUIT SECTION
--============================================================

Section(
	"FRUIT TRACKER",
	430
)

local FruitScanButton =
	Button(
		"SCAN WHOLE MAP NOW",
	458,
		function()

			FruitScanRequested =
				true
		end
	)

local FruitStatus =
	Status(
		"Fruit scanner: starting...",
		500
	)

local FruitTimerStatus =
	Status(
		"Next scan: 20s",
		525
	)

--============================================================
-- COORDINATES
--============================================================

Section(
	"COORDINATES",
	560
)

local CoordinateStatus =
	Status(
		"X: 0   Y: 0   Z: 0",
		588
	)

local FriendNameBox =
	InputBox(
		"Friend marker name",
		"",
		620
	)

local FriendXBox =
	InputBox(
		"Friend X",
		"",
		655
	)

local FriendYBox =
	InputBox(
		"Friend Y",
		"",
		690
	)

local FriendZBox =
	InputBox(
		"Friend Z",
		"",
		725
	)

local FriendColorBox =
	InputBox(
		"Color: R,G,B  e.g. 0,170,255",
		"0,170,255",
		760
	)

local AddFriendButton =
	Button(
		"ADD FRIEND MARKER",
		795,
		function()

			local name =
				FriendNameBox.Text

			local x =
				tonumber(
					FriendXBox.Text
				)

			local y =
				tonumber(
					FriendYBox.Text
				)

			local z =
				tonumber(
					FriendZBox.Text
				)

			if name == ""
				or not x
				or not y
				or not z then

				return
			end

			local r, g, b =
				string.match(
					FriendColorBox.Text,
					"(%d+)%s*,%s*(%d+)%s*,%s*(%d+)"
				)

			r = tonumber(r) or 0
			g = tonumber(g) or 170
			b = tonumber(b) or 255

			local color =
				Color3.fromRGB(
					math.clamp(r, 0, 255),
					math.clamp(g, 0, 255),
					math.clamp(b, 0, 255)
				)

			CreateFriendMarker(
				name,
				Vector3.new(
					x,
					y,
					z
				),
				color
			)

		end
	)

local ClearFriendsButton =
	Button(
		"CLEAR FRIEND MARKERS",
	835,
		function()

			for name, part in pairs(
				FriendMarkers
			) do

				if part then
					part:Destroy()
				end

				FriendMarkers[name] =
					nil
			end
		end
	)

local CoordinateHelp =
	Status(
		"Enter world X/Y/Z coordinates to create a marker.",
		875
	)

--============================================================
-- HALLOWEEN PUMPKIN FARM
--============================================================

local PUMPKIN_TWEEN_SPEED = 45 -- studs per second
local PumpkinFarmEnabled = false
local PumpkinFarmRunning = false
local PumpkinFarmTween = nil

Section(
	"HALLOWEEN",
	920
)

local PumpkinSpeedBox =
	InputBox(
		"Tween speed (studs per second)",
		tostring(PUMPKIN_TWEEN_SPEED),
		948
)

PumpkinSpeedBox.FocusLost:Connect(function()
	local value = tonumber(PumpkinSpeedBox.Text)
	if value then
		PUMPKIN_TWEEN_SPEED = math.clamp(value, 5, 500)
		PumpkinSpeedBox.Text = tostring(PUMPKIN_TWEEN_SPEED)
	else
		PumpkinSpeedBox.Text = tostring(PUMPKIN_TWEEN_SPEED)
	end
end)

local PumpkinStatus =
	Status(
		"Pumpkin farm: OFF",
		980
)

local PumpkinButton
local StopPumpkinFarm

local function GetPumpkinPart(target)
	if not target then return nil end
	if target:IsA("BasePart") then
		return target
	elseif target:IsA("Model") then
		return target.PrimaryPart or target:FindFirstChildWhichIsA("BasePart", true)
	end
	return nil
end

local function FindPumpkinTarget()
	if not RootPart then return nil end

	local closestTarget = nil
	local closestPart = nil
	local closestDistance = math.huge

	-- Choose the closest valid Pumpkin, not whichever one happens to be
	-- returned first by GetDescendants().
	for _, object in ipairs(Workspace:GetDescendants()) do
		if object.Name == "Pumpkin" and (object:IsA("BasePart") or object:IsA("Model")) then
			local part = GetPumpkinPart(object)
			if part and part:IsDescendantOf(Workspace) then
				local distance = (part.Position - RootPart.Position).Magnitude
				if distance < closestDistance then
					closestDistance = distance
					closestTarget = object
					closestPart = part
				end
			end
		end
	end

	return closestTarget, closestPart, closestDistance
end

local function GetPumpkinApproachCFrame(target, targetPart)
	if not targetPart then return nil end

	local targetPosition = targetPart.Position
	local targetHeight = targetPart.Size.Y

	if target:IsA("Model") then
		local _, bounds = target:GetBoundingBox()
		targetHeight = bounds.Y
		targetPosition = target:GetPivot().Position
	end

	-- Look for a nearby surface above the pumpkin. If the pumpkin is below
	-- the map/floor, approach above that surface instead of tweening below it.
	local rayParams = RaycastParams.new()
	rayParams.FilterType = Enum.RaycastFilterType.Exclude
	rayParams.FilterDescendantsInstances = {Character, target}
	rayParams.IgnoreWater = true

	local rayOrigin = targetPosition + Vector3.new(0, 20, 0)
	local rayResult = Workspace:Raycast(rayOrigin, Vector3.new(0, -120, 0), rayParams)

	local safeY = targetPosition.Y + (targetHeight * 0.5) + 4
	if rayResult and rayResult.Position.Y > targetPosition.Y + 6 then
		safeY = math.max(safeY, rayResult.Position.Y + 4)
	end

	-- Keep the root above a minimum world height as a final safety guard.
	safeY = math.max(safeY, Workspace.FallenPartsDestroyHeight + 25)

	return CFrame.new(targetPosition.X, safeY, targetPosition.Z)
end

local function GetFirstBackpackTool()
	if not Character then return nil end
	local backpack = Player:FindFirstChildOfClass("Backpack")
	if not backpack then return nil end

	-- The first Tool in Backpack is used as hotbar slot 1.
	for _, item in ipairs(backpack:GetChildren()) do
		if item:IsA("Tool") then
			return item
		end
	end
	return nil
end

local function PumpkinStillExists(target)
	return target ~= nil
		and target.Parent ~= nil
		and target:IsDescendantOf(Workspace)
end

StopPumpkinFarm = function()
	PumpkinFarmEnabled = false
	if PumpkinFarmTween then
		PumpkinFarmTween:Cancel()
		PumpkinFarmTween = nil
	end
	if PumpkinButton then
		PumpkinButton.Text = "PUMPKIN FARM  •  OFF"
	end
	if PumpkinStatus then
		PumpkinStatus.Text = "Pumpkin farm: OFF"
	end
end

local function RunPumpkinFarm()
	if PumpkinFarmRunning then return end
	PumpkinFarmRunning = true

	task.spawn(function()
		while PumpkinFarmEnabled do
			local target, targetPart = FindPumpkinTarget()

			if not target then
				PumpkinStatus.Text = "Pumpkin farm: waiting for Pumpkin..."
				task.wait(0.5)
				continue
			end

			if not RootPart or not Humanoid or Humanoid.Health <= 0 then
				PumpkinStatus.Text = "Pumpkin farm: waiting for character..."
				task.wait(0.5)
				continue
			end

			-- Recalculate the closest target immediately before moving.
			target, targetPart = FindPumpkinTarget()
			if not target or not targetPart then
				task.wait(0.2)
				continue
			end

			local destination = GetPumpkinApproachCFrame(target, targetPart)
			if not destination then
				task.wait(0.25)
				continue
			end

			local distance = (RootPart.Position - destination.Position).Magnitude
			local duration = math.max(distance / PUMPKIN_TWEEN_SPEED, 0.08)

			PumpkinStatus.Text = string.format(
				"Pumpkin farm: moving to closest Pumpkin (%.0f studs)",
				distance
			)

			-- Move horizontally at the character's current height first, then
			-- move vertically to the safe approach height. This avoids a
			-- diagonal tween cutting through the map/floor.
			local horizontalDestination = CFrame.new(
				destination.Position.X,
				RootPart.Position.Y,
				destination.Position.Z
			)
			local horizontalDistance = (RootPart.Position - horizontalDestination.Position).Magnitude
			local horizontalDuration = math.max(horizontalDistance / PUMPKIN_TWEEN_SPEED, 0.05)

			PumpkinFarmTween = TweenService:Create(
				RootPart,
				TweenInfo.new(horizontalDuration, Enum.EasingStyle.Linear),
				{CFrame = horizontalDestination}
			)
			PumpkinFarmTween:Play()

			local horizontalDone = false
			local horizontalConnection
			horizontalConnection = PumpkinFarmTween.Completed:Connect(function(state)
				horizontalDone = state == Enum.PlaybackState.Completed
			end)

			while PumpkinFarmEnabled and PumpkinStillExists(target) and not horizontalDone do
				task.wait(0.05)
			end
			if horizontalConnection then horizontalConnection:Disconnect() end
			if PumpkinFarmTween then
				PumpkinFarmTween:Cancel()
				PumpkinFarmTween = nil
			end

			if not PumpkinFarmEnabled or not PumpkinStillExists(target) then
				continue
			end

			local verticalDistance = math.abs(destination.Position.Y - RootPart.Position.Y)
			local verticalDuration = math.max(verticalDistance / PUMPKIN_TWEEN_SPEED, 0.05)
			PumpkinFarmTween = TweenService:Create(
				RootPart,
				TweenInfo.new(verticalDuration, Enum.EasingStyle.Linear),
				{CFrame = destination}
			)
			PumpkinFarmTween:Play()

			local completed = false
			local connection
			connection = PumpkinFarmTween.Completed:Connect(function(state)
				completed = state == Enum.PlaybackState.Completed
			end)

			while PumpkinFarmEnabled and PumpkinStillExists(target) and not completed do
				task.wait(0.05)
			end
			if connection then connection:Disconnect() end
			if PumpkinFarmTween then
				PumpkinFarmTween:Cancel()
				PumpkinFarmTween = nil
			end

			if not PumpkinFarmEnabled then break end
			if not PumpkinStillExists(target) then
				task.wait(0.2)
				continue
			end

			-- Press hotbar key 1 exactly once after reaching this Pumpkin.
			-- Do not send Backquote (`), which toggles the Roblox backpack.
			PumpkinStatus.Text = "Pumpkin farm: selecting slot 1..."
			local keyPressed = pcall(function()
				VirtualInputManager:SendKeyEvent(true, Enum.KeyCode.One, false, game)
				 task.wait(0.05)
				VirtualInputManager:SendKeyEvent(false, Enum.KeyCode.One, false, game)
			end)

			-- If virtual key input is unavailable, equip one Tool directly as a
			-- fallback. This does not repeatedly equip it during the attack loop.
			if not keyPressed then
				local fallbackTool = GetFirstBackpackTool()
				if fallbackTool and Humanoid then
					pcall(function()
						Humanoid:EquipTool(fallbackTool)
					end)
				end
			end

			task.wait(0.12)

			local tool = Character and Character:FindFirstChildOfClass("Tool")
			if not tool then
				PumpkinStatus.Text = "Pumpkin farm: slot 1 has no Tool"
				task.wait(0.3)
			else
				PumpkinStatus.Text = "Pumpkin farm: M1 attacking Pumpkin..."
				-- Keep M1 active while this Pumpkin exists. Slot 1 is NOT pressed
				-- again, and the Tool is NOT re-equipped on every attack.
				while PumpkinFarmEnabled and PumpkinStillExists(target) do
					pcall(function()
						tool:Activate()
					end)
					task.wait(0.18)
				end
			end
		end

		PumpkinFarmRunning = false
		if not PumpkinFarmEnabled then
			PumpkinStatus.Text = "Pumpkin farm: OFF"
		end
	end)
end

PumpkinButton =
	Button(
		"PUMPKIN FARM  •  OFF",
		1010,
		function()
			if PumpkinFarmEnabled then
				StopPumpkinFarm()
			else
				PumpkinFarmEnabled = true
				PumpkinButton.Text = "PUMPKIN FARM  •  ON"
				PumpkinStatus.Text = "Pumpkin farm: searching..."
				RunPumpkinFarm()
			end
		end
	)

--============================================================
-- GUI DRAG
--============================================================

local Dragging = false
local DragStart
local StartPosition

Header.InputBegan:Connect(
	function(input)

		if input.UserInputType == Enum.UserInputType.MouseButton1
			or input.UserInputType == Enum.UserInputType.Touch then

			Dragging = true
			DragStart = input.Position
			StartPosition = Main.Position
		end
	end
)

UserInputService.InputChanged:Connect(
	function(input)

		if not Dragging then
			return
		end

		if input.UserInputType == Enum.UserInputType.MouseMovement
			or input.UserInputType == Enum.UserInputType.Touch then

			local delta =
				input.Position -
				DragStart

			Main.Position =
				UDim2.new(
					StartPosition.X.Scale,
					StartPosition.X.Offset +
						delta.X,

					StartPosition.Y.Scale,
					StartPosition.Y.Offset +
						delta.Y
				)
		end
	end
)

UserInputService.InputEnded:Connect(
	function(input)

		if input.UserInputType == Enum.UserInputType.MouseButton1
			or input.UserInputType == Enum.UserInputType.Touch then

			Dragging = false
		end
	end
)

--============================================================
-- COLLAPSE
--============================================================

Minimize.MouseButton1Click:Connect(
	function()

		MenuCollapsed =
			not MenuCollapsed

		Content.Visible =
			not MenuCollapsed

		if MenuCollapsed then

			Main.Size =
				UDim2.fromOffset(
					390,
					72
				)

			Minimize.Text = "+"

		else

			Main.Size =
				UDim2.fromOffset(
					390,
					640
				)

			Minimize.Text = "−"
		end
	end
)

Close.Activated:Connect(function()
	MenuOpen = false
	Main.Visible = false
	ReopenButton.Visible = IsMobile
end)

--============================================================
-- FRUIT HELPERS
--============================================================

local function GetFruitName(name)

	local lower =
		string.lower(name)

	for lookup, displayName in pairs(
		FruitLookup
	) do

		if lower == lookup then
			return displayName
		end

		if string.find(
			lower,
			lookup,
			1,
			true
		) then

			return displayName
		end
	end

	return nil
end

local function FindFruit(object)

	local current = object

	while current
		and current ~= Workspace do

		local fruitName =
			GetFruitName(
				current.Name
			)

		if fruitName then
			return current, fruitName
		end

		current =
			current.Parent
	end

	return nil
end

local function GetPart(object)

	if object:IsA("BasePart") then
		return object
	end

	if object:IsA("Model") then

		if object.PrimaryPart then
			return object.PrimaryPart
		end

		return object:FindFirstChildWhichIsA(
			"BasePart",
			true
		)
	end

	if object:IsA("Tool") then

		return object:FindFirstChild(
			"Handle"
		) or object:FindFirstChildWhichIsA(
			"BasePart",
			true
		)
	end

	return nil
end

--============================================================
-- FRUIT MARKER
--============================================================

local function RemoveFruitMarker(object)

	local marker =
		FruitMarkers[object]

	if marker then

		if marker.Gui then
			marker.Gui:Destroy()
		end

		FruitMarkers[object] =
			nil
	end

	KnownFruits[object] =
		nil
end

local function AddFruitMarker(
	object,
	fruitName
)

	if not object
		or not object.Parent then
		return
	end

	if KnownFruits[object] then
		return
	end

	local part =
		GetPart(object)

	if not part then
		return
	end

	local count = 0

	for _ in pairs(
		FruitMarkers
	) do
		count += 1
	end

	if count >= MAX_FRUIT_MARKERS then
		return
	end

	KnownFruits[object] =
		true

	local billboard =
		Instance.new("BillboardGui")

	billboard.Name =
		"HakNoteFruitMarker"

	billboard.Adornee =
		part

	billboard.Size =
		UDim2.fromOffset(
			240,
			65
		)

	billboard.StudsOffset =
		Vector3.new(
			0,
			4,
			0
		)

	billboard.AlwaysOnTop = true
	billboard.MaxDistance = 100000

	billboard.Parent = Gui

	local background =
		Instance.new("Frame")

	background.Size =
		UDim2.fromScale(
			1,
			1
		)

	background.BackgroundColor3 =
		Color3.fromRGB(
			13,
			15,
			20
		)

	background.BackgroundTransparency =
		0.05

	background.BorderSizePixel = 0
	background.Parent = billboard

	local corner =
		Instance.new("UICorner")

	corner.CornerRadius =
		UDim.new(0, 9)

	corner.Parent = background

	local icon =
		Instance.new("ImageLabel")

	icon.Size =
		UDim2.fromOffset(
			45,
			45
		)

	icon.Position =
		UDim2.fromOffset(
			8,
			10
		)

	icon.BackgroundTransparency = 1
	icon.Image = LOGO_ID
	icon.Parent = background

	local label =
		Instance.new("TextLabel")

	label.Size =
		UDim2.new(
			1,
			-60,
			0,
			28
		)

	label.Position =
		UDim2.fromOffset(
			58,
			8
		)

	label.BackgroundTransparency = 1
	label.Text = fruitName

	label.TextColor3 =
		Color3.fromRGB(
			245,
			245,
			250
		)

	label.Font =
		Enum.Font.GothamBlack

	label.TextSize = 14
	label.TextXAlignment =
		Enum.TextXAlignment.Left

	label.Parent = background

	local distance =
		Instance.new("TextLabel")

	distance.Size =
		UDim2.new(
			1,
			-60,
			0,
			20
		)

	distance.Position =
		UDim2.fromOffset(
			58,
			36
		)

	distance.BackgroundTransparency = 1

	distance.Text =
		"DETECTED"

	distance.TextColor3 =
		Color3.fromRGB(
			115,
			205,
			255
		)

	distance.Font =
		Enum.Font.GothamBold

	distance.TextSize = 10
	distance.TextXAlignment =
		Enum.TextXAlignment.Left

	distance.Parent = background

	FruitMarkers[object] = {

		Gui = billboard,

		Part = part,

		Distance = distance,

		Name = fruitName
	}
end

--============================================================
-- FRUIT SCAN
--============================================================

local function ProcessFruitObject(
	object
)

	if not object
		or not object.Parent then
		return
	end

	local fruitObject,
		fruitName =
		FindFruit(object)

	if fruitObject
		and fruitName then

		AddFruitMarker(
			fruitObject,
			fruitName
		)
	end
end

Workspace.DescendantAdded:Connect(
	function(object)

		task.defer(
			function()
				ProcessFruitObject(
					object
				)
			end
		)
	end
)

Workspace.DescendantRemoving:Connect(
	function(object)

		RemoveFruitMarker(
			object
		)
	end
)

local function FullFruitScan()

	if FruitScanRunning then

		FruitScanRequested =
			true

		return
	end

	FruitScanRunning = true
	FruitScanRequested = false

	task.spawn(
		function()

			local objects =
				Workspace:GetDescendants()

			local total =
				#objects

			for index, object in ipairs(
				objects
			) do

				ProcessFruitObject(
					object
				)

				if index %
					FRUIT_SCAN_BATCH ==
					0 then

					FruitStatus.Text =
						string.format(
							"Scanning fruits... %d%%",
							math.floor(
								index /
								total *
								100
							)
						)

					task.wait()
				end
			end

			local count = 0

			for _ in pairs(
				FruitMarkers
			) do
				count += 1
			end

			FruitStatus.Text =
				"Fruit markers: " ..
				count

			FruitScanRunning =
				false

			if FruitScanRequested then

				task.defer(
					FullFruitScan
				)
			end
		end
	)
end

--============================================================
-- NPC FOLDER
--============================================================

local function GetNPCFolder()

	return Workspace:FindFirstChild(
		"Enemies"
	) or Workspace:FindFirstChild(
		"NPCs"
	)
end

--============================================================
-- NPC SEARCH
--============================================================

local function GetNearestNPC()

	if not RootPart then
		return nil, math.huge
	end

	local folder =
		GetNPCFolder()

	if not folder then
		return nil, math.huge
	end

	local closest
	local closestDistance =
		NPC_RANGE

	for _, npc in ipairs(
		folder:GetChildren()
	) do

		local humanoid =
			npc:FindFirstChildOfClass(
				"Humanoid"
			)

		local root =
			npc:FindFirstChild(
				"HumanoidRootPart"
			)

		if humanoid
			and root
			and humanoid.Health > 0 then

			local distance =
				(
					root.Position -
					RootPart.Position
				).Magnitude

			if distance <
				closestDistance then

				closest =
					npc

				closestDistance =
					distance
			end
		end
	end

	return closest,
		closestDistance
end

--============================================================
-- SMOOTH NPC MOVEMENT
--============================================================

local function UpdateNPC()

	if not NPCEnabled then
		return
	end

	if not RootPart
		or not Humanoid
		or Humanoid.Health <= 0 then
		return
	end

	local npc,
		distance =
		GetNearestNPC()

	CurrentNPC =
		npc

	if not npc then

		NPCStatus.Text =
			"NPC: NONE"

		return
	end

	local npcRoot =
		npc:FindFirstChild(
			"HumanoidRootPart"
		)

	if not npcRoot then
		return
	end

	NPCStatus.Text =
		string.format(
			"NPC: %s  •  %.0f studs",
			npc.Name,
			distance
		)

	-- Position above and slightly behind the NPC.
	local target =
		npcRoot.Position +
		Vector3.new(
			0,
			NPC_HOVER_HEIGHT,
			0
		) -
		(
			npcRoot.CFrame.LookVector *
			NPC_HOVER_DISTANCE
		)

	-- Smooth movement rather than
	-- hard teleporting every frame.
	local newPosition =
		RootPart.Position:Lerp(
			target,
			NPC_MOVE_SMOOTHNESS
		)

	local look =
		CFrame.lookAt(
			newPosition,
			npcRoot.Position
		)

	RootPart.CFrame =
		look
end

--============================================================
-- NEARBY RADAR
--============================================================

local function GetRadarTarget(part)
	local current = part
	while current and current ~= Workspace do
		if current:IsA("Model") or current:IsA("Tool") then
			if Character and current == Character then
				return nil
			end
			return current
		end
		current = current.Parent
	end
	if part:IsA("BasePart") and not (Character and part:IsDescendantOf(Character)) then
		return part
	end
	return nil
end

local function GetTargetPosition(target)
	if target:IsA("BasePart") then
		return target.Position
	elseif target:IsA("Model") then
		return target:GetPivot().Position
	elseif target:IsA("Tool") then
		local handle = target:FindFirstChild("Handle") or target:FindFirstChildWhichIsA("BasePart", true)
		return handle and handle.Position or nil
	end
	return nil
end

ClearESP = function()
	for target, highlight in pairs(ESPHighlights) do
		if highlight then highlight:Destroy() end
		ESPHighlights[target] = nil
	end
end

local function ApplyNameESP()
	ClearESP()
	local search = string.lower(string.gsub(ESPNameBox.Text, "^%s*(.-)%s*$", "%1"))
	if search == "" then
		RadarPanelTitle.Text = "20-STUD RADAR  •  ENTER A NAME"
		return
	end

	local matched = {}
	for _, descendant in ipairs(Workspace:GetDescendants()) do
		if descendant:IsA("BasePart") then
			local target = GetRadarTarget(descendant)
			if target and not matched[target] and string.find(string.lower(target.Name), search, 1, true) then
				matched[target] = true
				local adornee = target
				if target:IsA("Tool") then
					adornee = target:FindFirstChild("Handle") or target:FindFirstChildWhichIsA("BasePart", true)
				end
				if not adornee then
					continue
				end
				local highlight = Instance.new("Highlight")
				highlight.Name = "HakNoteNameESP"
				highlight.Adornee = adornee
				highlight.FillColor = Color3.fromRGB(35, 170, 255)
				highlight.OutlineColor = Color3.fromRGB(235, 245, 255)
				highlight.FillTransparency = 0.65
				highlight.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
				highlight.Parent = Gui
				ESPHighlights[target] = highlight
			end
		end
	end

	local count = 0
	for _ in pairs(ESPHighlights) do count += 1 end
	RadarPanelTitle.Text = string.format("20-STUD RADAR  •  ESP: %d MATCH(ES)", count)
end

ESPButton.MouseButton1Click:Connect(ApplyNameESP)

local function ScanNearby()
	if not RootPart then return end
	table.clear(NearbyObjects)

	local nearbyParts = Workspace:GetPartBoundsInRadius(RootPart.Position, NEARBY_RANGE)
	for _, part in ipairs(nearbyParts) do
		local target = GetRadarTarget(part)
		if target then
			local position = GetTargetPosition(target)
			if position then
				local distance = (position - RootPart.Position).Magnitude
				if distance <= NEARBY_RANGE then
					NearbyObjects[target] = {
						Name = target.Name,
						Distance = distance
					}
				end
			end
		end
	end

	local entries = {}
	for target, info in pairs(NearbyObjects) do
		if target and target.Parent then table.insert(entries, info) end
	end
	table.sort(entries, function(a, b) return a.Distance < b.Distance end)

	local lines = {}
	for index, info in ipairs(entries) do
		table.insert(lines, string.format("%02d. %s  •  %.1f studs", index, info.Name, info.Distance))
	end
	RadarListText.Text = #lines > 0 and table.concat(lines, "\\n") or "No objects within 20 studs."
	RadarListText.Size = UDim2.new(1, -12, 0, math.max(24, #lines * 19))
	RadarList.CanvasSize = UDim2.fromOffset(0, math.max(0, #lines * 19 + 10))
	RadarStatus.Text = "Nearby objects: " .. tostring(#entries)
end

-- FLIGHT
--============================================================

function DestroyFlight()

	if FlightVelocity then
		FlightVelocity:Destroy()
		FlightVelocity = nil
	end

	if FlightOrientation then
		FlightOrientation:Destroy()
		FlightOrientation = nil
	end

	if FlightAttachment then
		FlightAttachment:Destroy()
		FlightAttachment = nil
	end

	if Humanoid then
		Humanoid.AutoRotate = true
	end
end

function CreateFlight()

	if not RootPart
		or not Humanoid then
		return
	end

	DestroyFlight()

	FlightAttachment =
		Instance.new("Attachment")

	FlightAttachment.Parent =
		RootPart

	FlightVelocity =
		Instance.new("LinearVelocity")

	FlightVelocity.Attachment0 =
		FlightAttachment

	FlightVelocity.RelativeTo =
		Enum.ActuatorRelativeTo.World

	FlightVelocity.VelocityConstraintMode =
		Enum.VelocityConstraintMode.Vector

	FlightVelocity.MaxForce =
		math.huge

	FlightVelocity.Parent =
		RootPart

	FlightOrientation =
		Instance.new("AlignOrientation")

	FlightOrientation.Attachment0 =
		FlightAttachment

	FlightOrientation.Mode =
		Enum.OrientationAlignmentMode.OneAttachment

	FlightOrientation.MaxTorque =
		math.huge

	FlightOrientation.Responsiveness =
		15

	FlightOrientation.Parent =
		RootPart

	Humanoid.AutoRotate = false
end

local function UpdateFlight()

	if not FlightEnabled
		or not RootPart
		or not Humanoid then
		return
	end

	if not FlightVelocity
		or not FlightVelocity.Parent then

		CreateFlight()
	end

	local camera =
		Workspace.CurrentCamera

	if not camera then
		return
	end

	local direction =
		Vector3.zero

	local forward =
		camera.CFrame.LookVector

	local right =
		camera.CFrame.RightVector

	if IsMobile then
		-- Use Roblox's mobile thumbstick for horizontal movement.
		direction += Humanoid.MoveDirection
		if MobileAscend then direction += Vector3.yAxis end
		if MobileDescend then direction -= Vector3.yAxis end
	else
		if UserInputService:IsKeyDown(Enum.KeyCode.W) then direction += forward end
		if UserInputService:IsKeyDown(Enum.KeyCode.S) then direction -= forward end
		if UserInputService:IsKeyDown(Enum.KeyCode.D) then direction += right end
		if UserInputService:IsKeyDown(Enum.KeyCode.A) then direction -= right end
		if UserInputService:IsKeyDown(Enum.KeyCode.Space) then direction += Vector3.yAxis end
		if UserInputService:IsKeyDown(Enum.KeyCode.C)
			or UserInputService:IsKeyDown(Enum.KeyCode.LeftControl) then
			direction -= Vector3.yAxis
		end
	end

	if direction.Magnitude > 0 then
		direction =
			direction.Unit
	end

	FlightVelocity.VectorVelocity =
		direction *
		FLIGHT_SPEED

	local flatLook =
		Vector3.new(
			forward.X,
			0,
			forward.Z
		)

	if flatLook.Magnitude > 0.01 then

		FlightOrientation.CFrame =
			CFrame.lookAt(
				Vector3.zero,
				flatLook.Unit
			)
	end
end

--============================================================
-- FRIEND MARKERS
--============================================================

function CreateFriendMarker(
	name,
	position,
	color
)

	if FriendMarkers[name] then

		FriendMarkers[name]:Destroy()

		FriendMarkers[name] =
			nil
	end

	local part =
		Instance.new("Part")

	part.Name =
		"HakNoteFriend_" ..
		name

	part.Size =
		Vector3.new(
			1,
			1,
			1
		)

	part.Position =
		position

	part.Anchored = true
	part.CanCollide = false
	part.CanTouch = false
	part.CanQuery = false
	part.Transparency = 1

	part.Parent =
		Workspace

	local billboard =
		Instance.new("BillboardGui")

	billboard.Size =
		UDim2.fromOffset(
			220,
			70
		)

	billboard.StudsOffset =
		Vector3.new(
			0,
			4,
			0
		)

	billboard.AlwaysOnTop = true
	billboard.Parent = part

	local background =
		Instance.new("Frame")

	background.Size =
		UDim2.fromScale(
			1,
			1
		)

	background.BackgroundColor3 =
		Color3.fromRGB(
			14,
			16,
			21
		)

	background.BackgroundTransparency =
		0.08

	background.BorderSizePixel = 0
	background.Parent = billboard

	local corner =
		Instance.new("UICorner")

	corner.CornerRadius =
		UDim.new(
			0,
			9
		)

	corner.Parent =
		background

	local label =
		Instance.new("TextLabel")

	label.Size =
		UDim2.new(
			1,
			-12,
			0,
			28
		)

	label.Position =
		UDim2.fromOffset(
			6,
			5
		)

	label.BackgroundTransparency = 1

	label.Text =
		"●  " .. name

	label.TextColor3 =
		color

	label.Font =
		Enum.Font.GothamBlack

	label.TextSize = 15
	label.Parent = background

	local coords =
		Instance.new("TextLabel")

	coords.Size =
		UDim2.new(
			1,
			-12,
			0,
			24
		)

	coords.Position =
		UDim2.fromOffset(
			6,
			35
		)

	coords.BackgroundTransparency = 1

	coords.Text =
		string.format(
			"%d, %d, %d",
			math.round(position.X),
			math.round(position.Y),
			math.round(position.Z)
		)

	coords.TextColor3 =
		Color3.fromRGB(
			205,
			210,
			220
		)

	coords.Font =
		Enum.Font.GothamBold

	coords.TextSize = 10
	coords.Parent = background

	FriendMarkers[name] =
		part
end

--============================================================
-- INPUT
--============================================================

UserInputService.InputBegan:Connect(
	function(input, processed)

		if input.KeyCode ==
			MENU_KEY then

			MenuOpen =
				not MenuOpen

			Main.Visible =
				MenuOpen

			return
		end

		if WaitingForFlightKey then

			if input.UserInputType ==
				Enum.UserInputType.Keyboard
				and input.KeyCode ~=
				Enum.KeyCode.Unknown
				and input.KeyCode ~=
				MENU_KEY then

				FLIGHT_KEY =
					input.KeyCode

				WaitingForFlightKey =
					false

				FlightKeyButton.Text =
					"FLIGHT KEY  •  " ..
					FLIGHT_KEY.Name
			end

			return
		end

		if processed then
			return
		end

		if input.KeyCode ==
			FLIGHT_KEY then

			FlightEnabled =
				not FlightEnabled

			FlightTouchControls.Visible = IsMobile and FlightEnabled

			if FlightEnabled then
				CreateFlight()
			else
				DestroyFlight()
			end
		end
	end
)

--============================================================
-- MAIN LOOP
--============================================================

RunService.Heartbeat:Connect(
	function(dt)

		-- Flight
		UpdateFlight()

		-- NPC scan
		NPCScanTimer += dt

		if NPCScanTimer >=
			NPC_SCAN_RATE then

			NPCScanTimer = 0

			if NPCEnabled then
				UpdateNPC()
			end
		end

		-- Nearby radar
		NearbyTimer += dt

		if NearbyTimer >=
			NEARBY_SCAN_RATE then

			NearbyTimer = 0

			if RadarEnabled then
				ScanNearby()
			end
		end

		-- Fruit marker updates
		MarkerTimer += dt

		if MarkerTimer >=
			MARKER_UPDATE_RATE then

			MarkerTimer = 0

			for object, marker in pairs(
				FruitMarkers
			) do

				if not object
					or not object.Parent then

					RemoveFruitMarker(
						object
					)

				else

					local part =
						GetPart(object)

					if not part then

						RemoveFruitMarker(
							object
						)

					else

						marker.Gui.Adornee =
							part

						if RootPart then

							local distance =
								(
									part.Position -
									RootPart.Position
								).Magnitude

							marker.Distance.Text =
								string.format(
									"● %.0f STUDS",
									distance
								)
						end
					end
				end
			end
		end

		-- Fruit timer
		FruitScanTimer += dt

		if FruitScanTimer >=
			FRUIT_SCAN_INTERVAL then

			FruitScanTimer = 0

			FruitScanRequested =
				true
		end

		if FruitScanRequested
			and not FruitScanRunning then

			FruitScanRequested =
				false

			task.defer(
				FullFruitScan
			)
		end

		-- Coordinates
		if RootPart then

			local p =
				RootPart.Position

			CoordinateStatus.Text =
				string.format(
					"X: %.0f   Y: %.0f   Z: %.0f",
					p.X,
					p.Y,
					p.Z
				)
		end

		-- Button state
		if FlightEnabled then

			FlightButton.Text =
				"FLIGHT  •  ON"

			FlightButton.BackgroundColor3 =
				Color3.fromRGB(
					35,
					90,
					120
				)

		else

			FlightButton.Text =
				"FLIGHT  •  OFF"

			FlightButton.BackgroundColor3 =
				Color3.fromRGB(
					31,
					34,
					42
				)
		end

		if NPCEnabled then

			NPCButton.Text =
				"NPC LOCK-ON  •  ON"

			NPCButton.BackgroundColor3 =
				Color3.fromRGB(
					45,
					95,
					70
				)

		else

			NPCButton.Text =
				"NPC LOCK-ON  •  OFF"

			NPCButton.BackgroundColor3 =
				Color3.fromRGB(
					31,
					34,
					42
				)
		end

		FruitTimerStatus.Text =
			"Next full scan: " ..
			math.max(
				0,
				math.ceil(
					FRUIT_SCAN_INTERVAL -
					FruitScanTimer
				)
			) ..
			"s"
	end
)

--============================================================
-- STARTUP
--============================================================

task.defer(
	FullFruitScan
)

print(
	"===================================="
)

print(
	"          HAK NOTE LOADED"
)

print(
	"===================================="
)

print(
	"Menu: `"
)

print(
	"Flight: " ..
		FLIGHT_KEY.Name
)

print(
	"Fruit scan: 20 seconds"
)

print(
	"Nearby radar: 20 studs"
)

print(
	"NPC lock-on: ready"
)

print(
	"Coordinate markers: ready"
)

print(
	"===================================="
)
