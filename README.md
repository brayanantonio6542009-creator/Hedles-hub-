if not game:IsLoaded() then game.Loaded:Wait() end

local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
_G.Mynxx_Ready = false   -- indica que o script ainda não está totalmente carregado
UIS = game:GetService("UserInputService")
RunService = game:GetService("RunService")
Stats = game:GetService("Stats")
TweenService = game:GetService("TweenService")
HttpService = game:GetService("HttpService")
ReplicatedStorage = game:GetService("ReplicatedStorage")
Workspace = game:GetService("Workspace")
Lighting = game:GetService("Lighting")
TeleportService = game:GetService("TeleportService")
CoreGui = game:GetService("CoreGui")



-- Fix spam: CraftingMachineController + Forge/EmitModule protected assets
_chargeSoundPart = nil
function ensureCraftingMachineParts()
    local cm = Workspace:FindFirstChild("CraftingMachine")
    if not cm then return false end
    local part = cm:FindFirstChild("ChargeSoundPart")
    if not part then
        part = _chargeSoundPart
        if not part or part.Parent ~= cm then
            part = Instance.new("Part")
            part.Name = "ChargeSoundPart"
            part.Anchored = true
            part.CanCollide = false
            part.CanQuery = false
            part.CanTouch = false
            part.Transparency = 1
            part.Size = Vector3.new(0.2, 0.2, 0.2)
            part.Archivable = false
            pcall(function() part.CFrame = cm:GetPivot() end)
            _chargeSoundPart = part
        end
        part.Parent = cm
    end
    return true
end

ensureCraftingMachineParts()
Workspace.ChildAdded:Connect(function(child)
    if child.Name == "CraftingMachine" then
        task.defer(ensureCraftingMachineParts)
    end
end)

task.spawn(function()
    for _ = 1, 30 do
        if ensureCraftingMachineParts() then break end
        task.wait(0.25)
    end
    local tickN = 0
    RunService.Heartbeat:Connect(function()
        tickN = tickN + 1
        if tickN % 30 == 0 then ensureCraftingMachineParts() end
    end)
end)

task.spawn(function()
    pcall(function()
        local controllers = ReplicatedStorage:WaitForChild("Controllers", 20)
        local modScript = controllers:WaitForChild("CraftingMachineController", 20)
        local ok, ctrl = pcall(require, modScript)
        if ok and type(ctrl) == "table" then
            for key, fn in pairs(ctrl) do
                if type(fn) == "function" and (key == "updateVFX" or tostring(key):lower():find("vfx")) then
                    local old = fn
                    ctrl[key] = function(...)
                        ensureCraftingMachineParts()
                        return pcall(old, ...)
                    end
                end
            end
        end
    end)
end)

pcall(function()
    if hookmetamethod then
        local oldIndex
        oldIndex = hookmetamethod(game, "__index", function(self, key)
            if key == "ChargeSoundPart" and typeof(self) == "Instance" and self:IsA("Model") and self.Name == "CraftingMachine" then
                local p = self:FindFirstChild("ChargeSoundPart")
                if not p then
                    ensureCraftingMachineParts()
                    p = self:FindFirstChild("ChargeSoundPart")
                end
                if p then return p end
            end
            return oldIndex(self, key)
        end)
    end
end)

-- Insta-reset state captured by the FireServer hook below.
-- GUID is the first arg of the balloon payload. Defaults to a randomly-generated
-- UUID so the payload is at least syntactically valid (an empty string was
-- silently rejected by the server -- that's why the reset wasn't firing). If you
-- have a known-good GUID, set `_G.Mynxx_RESET_GUID` before the script runs.
function _newGUID()
    local hex = "0123456789abcdef"
    local t = {}
    for i = 1, 32 do t[#t+1] = hex:sub(math.random(1,16), math.random(1,16)) end
    return table.concat(t, "", 1, 8) .. "-" .. table.concat(t, "", 9, 12)
        .. "-" .. table.concat(t, "", 13, 16) .. "-" .. table.concat(t, "", 17, 20)
        .. "-" .. table.concat(t, "", 21, 32)
end

function makeOneWay(plat)
    if not plat then return end
    local rsConn
    local lastY = nil
    rsConn = game:GetService("RunService").Stepped:Connect(function()
        if not plat or not plat.Parent then
            if rsConn then rsConn:Disconnect() end
            return
        end
        local char = game.Players.LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if hrp then
            local currentY = hrp.Position.Y
            if not lastY then lastY = currentY end
            
            local deltaY = currentY - lastY
            local isMovingUp = (hrp.AssemblyLinearVelocity.Y > 1) or (deltaY > 0.01 and deltaY < 5)
            
            if isMovingUp then
                plat.CanCollide = false
            else
                if currentY > plat.Position.Y + 0.1 then
                    plat.CanCollide = true
                else
                    plat.CanCollide = false
                end
            end
            
            lastY = currentY
        end
    end)
end

GUID = GUID or _G.Mynxx_RESET_GUID or _newGUID()
resetRemote = nil
instaResetCooldown = false

-- Reset seguro: não intercepta FireServer, não captura remotes e não envia payloads.
-- O reset abaixo usa somente a API local do Humanoid.
resetRemote = nil
instaResetCooldown = false

-- LPH_NO_VIRTUALIZE: real preprocessor directive under Luraph; harmless passthrough
-- everywhere else. Declared once globally so every heavy hot-path body can wrap
-- itself without each call site re-shimming.
if not LPH_NO_VIRTUALIZE then LPH_NO_VIRTUALIZE = function(fn) return fn end end

-- =====================================================================
-- LAZY-LOAD QUEUE
-- =====================================================================
_G.__MynxxLazyQ = _G.__MynxxLazyQ or {}
function LazyInit(name, fn)
    table.insert(_G.__MynxxLazyQ, { name = name, fn = LPH_NO_VIRTUALIZE(fn) })
end
task.defer(LPH_NO_VIRTUALIZE(function()
    while true do
        if #_G.__MynxxLazyQ > 0 then
            local item = table.remove(_G.__MynxxLazyQ, 1)
            local ok, err = pcall(item.fn)
            if not ok then warn("[Mynxx Lazy]", item.name, err) end
            task.wait(0.08)
        else
            task.wait(0.5)
        end
    end
end))

-- Persistent panel visibility store that survives script re-executes within the same Roblox session
_G._MynxxPanelVis = _G._MynxxPanelVis or {}

_G.lazyUIs = {}
_G.addLazyUI = function(element, targetVis, isScreenGui, panelName)
    if element then
        if isScreenGui then
            element.Enabled = false
        else
            element.Visible = false
        end
        table.insert(_G.lazyUIs, {element = element, targetVis = targetVis, isScreenGui = isScreenGui, cancelled = false, panelName = panelName})
    end
end
_G.cancelLazyUI = function(element)
    for _, item in ipairs(_G.lazyUIs) do
        if item.element == element then
            item.cancelled = true
            break
        end
    end
end
task.delay(4.0, function()
    for _, item in ipairs(_G.lazyUIs) do
        if item.element and not item.cancelled then
            local vis
            if item.panelName then
                -- Priority: _G store (survives re-exec) > Config.Visibilities > default true
                local fromG = _G._MynxxPanelVis[item.panelName]
                if fromG ~= nil then
                    vis = fromG
                elseif Config and Config.Visibilities then
                    local saved = Config.Visibilities[item.panelName]
                    if saved ~= nil then vis = saved else vis = true end
                else
                    vis = item.targetVis
                end
            else
                vis = item.targetVis
            end
            if item.isScreenGui then
                pcall(function() item.element.Enabled = vis end)
            elseif item.element.Parent then
                pcall(function() item.element.Visible = vis end)
            end
        end
    end
end)

player = Players.LocalPlayer
playerGui = player:WaitForChild("PlayerGui")
pcall(function() Players.RespawnTime = 0 end)

-- GLOBAL SCALE SYSTEM (Mobile Friendly Auto-Scaling)
GlobalUIScaleVal = 1
-- weak keys: transient ScreenGuis (alerts, popups) drop out of this map once
-- destroyed instead of piling up dead entries the rescale loop keeps touching
scaledGuis = setmetatable({}, { __mode = "k" })

function getGlobalScale()
    return GlobalUIScaleVal
end

function updateAllGuisScale(newScale)
    GlobalUIScaleVal = newScale
    for sg, master in pairs(scaledGuis) do
        pcall(function()
            if sg and sg.Parent and master and master.Parent then
                local scaleObj = master:FindFirstChild("Mynxx_GlobalScale")
                if scaleObj then
                    scaleObj.Scale = newScale
                end
                master.Size = UDim2.new(1 / newScale, 0, 1 / newScale, 0)
            end
        end)
    end
end

function registerScreenGui(sg)
    local master = sg:FindFirstChild("Mynxx_MasterFrame")
    if not master then
        master = Instance.new("Frame")
        master.Name = "Mynxx_MasterFrame"
        master.BackgroundTransparency = 1
        master.BorderSizePixel = 0
        master.Parent = sg
        
        local scaleObj = Instance.new("UIScale")
        scaleObj.Name = "Mynxx_GlobalScale"
        scaleObj.Parent = master
    end
    scaledGuis[sg] = master
    
    pcall(function()
        local scaleObj = master:FindFirstChild("Mynxx_GlobalScale")
        if scaleObj then
            scaleObj.Scale = GlobalUIScaleVal
        end
        master.Size = UDim2.new(1 / GlobalUIScaleVal, 0, 1 / GlobalUIScaleVal, 0)
    end)
    return master
end

function recalculateScale()
    local cam = Workspace.CurrentCamera
    if not cam then return end
    local size = cam.ViewportSize
    local h = size.Y
    
    local isMobile = UIS.TouchEnabled
    local newScale = 1
    if isMobile then
        -- Mobile/Tablet scaling: aggressive down-scale to perfectly fit small landscape screens
        newScale = math.clamp(h / 1000, 0.40, 0.58)
    else
        -- PC/Console scaling: standard comfortable scaling
        newScale = math.clamp(h / 800, 0.65, 1.0)
    end
    updateAllGuisScale(newScale)
end

function setupCameraListener()
    if cameraConn then pcall(function() cameraConn:Disconnect() end) end
    local cam = Workspace.CurrentCamera
    if cam then
        cameraConn = cam:GetPropertyChangedSignal("ViewportSize"):Connect(recalculateScale)
        recalculateScale()
    end
end
Workspace:GetPropertyChangedSignal("CurrentCamera"):Connect(setupCameraListener)
task.spawn(setupCameraListener)

local old = playerGui:FindFirstChild("MynxxHub_V3"); if old then old:Destroy() end
gui_sg = Instance.new("ScreenGui"); gui_sg.Name = "MynxxHub_V3"; gui_sg.ResetOnSpawn = false; gui_sg.IgnoreGuiInset = true; gui_sg.DisplayOrder = 9999999; gui_sg.Parent = playerGui
gui = registerScreenGui(gui_sg)

-- SHARED TOGGLE STATE
ToggleState = {}
function regToggle(name, default)
    if not ToggleState[name] then ToggleState[name] = {value = default or false, listeners = {}} end
end
function getToggle(name) return ToggleState[name] and ToggleState[name].value or false end
function setToggle(name, val, skipNotify)
    regToggle(name)
    ToggleState[name].value = val
    if not skipNotify then
        for _, fn in ipairs(ToggleState[name].listeners) do pcall(fn, val) end
    end
end
function onToggleChanged(name, fn)
    regToggle(name); table.insert(ToggleState[name].listeners, fn)
end

-- GLOBAL CONFIG FORWARD DECLARATIONS (to avoid undefined/shadowing issues)
local Config

-- THEME (Modern sidebar UI)
Themes = {
    Light = {
        Background=Color3.fromRGB(28,28,30), MainBackground=Color3.fromRGB(22,22,24),
        SidebarBg=Color3.fromRGB(16,16,18), ContentBg=Color3.fromRGB(30,30,33),
        SidebarActive=Color3.fromRGB(55,55,60), SidebarActiveText=Color3.fromRGB(255,255,255),
        SidebarInactiveText=Color3.fromRGB(185,185,192),
        Panel=Color3.fromRGB(36,36,40), Row=Color3.fromRGB(44,44,48), RowHover=Color3.fromRGB(56,56,62),
        Accent=Color3.fromRGB(175,175,185), AccentLight=Color3.fromRGB(95,95,105),
        Green=Color3.fromRGB(175,175,185), Red=Color3.fromRGB(210,80,90), Red2=Color3.fromRGB(185,60,72),
        Text=Color3.fromRGB(255,255,255), Dim=Color3.fromRGB(155,155,163), Stroke=Color3.fromRGB(0,0,0),
        SoftButton=Color3.fromRGB(52,52,58), SoftButtonHover=Color3.fromRGB(66,66,73),
        SoftAccent=Color3.fromRGB(44,44,50), SoftAccentHover=Color3.fromRGB(56,56,63),
        ToggleOn=Color3.fromRGB(195,195,205), ToggleOff=Color3.fromRGB(42,42,47), ToggleOff2=Color3.fromRGB(42,42,47),
        ToggleKnobOn=Color3.fromRGB(22,22,24), ToggleKnobOff=Color3.fromRGB(105,105,113),
        InputBg=Color3.fromRGB(33,33,37), SliderBg=Color3.fromRGB(48,48,53), SliderFill=Color3.fromRGB(195,195,205),
        BlacklistHover=Color3.fromRGB(65,38,44), BlacklistLeave=Color3.fromRGB(40,30,34),
    },
    Dark = {
        Background=Color3.fromRGB(28,28,30), MainBackground=Color3.fromRGB(22,22,24),
        SidebarBg=Color3.fromRGB(16,16,18), ContentBg=Color3.fromRGB(30,30,33),
        SidebarActive=Color3.fromRGB(55,55,60), SidebarActiveText=Color3.fromRGB(255,255,255),
        SidebarInactiveText=Color3.fromRGB(185,185,192),
        Panel=Color3.fromRGB(36,36,40), Row=Color3.fromRGB(44,44,48), RowHover=Color3.fromRGB(56,56,62),
        Accent=Color3.fromRGB(175,175,185), AccentLight=Color3.fromRGB(95,95,105),
        Green=Color3.fromRGB(175,175,185), Red=Color3.fromRGB(210,80,90), Red2=Color3.fromRGB(185,60,72),
        Text=Color3.fromRGB(255,255,255), Dim=Color3.fromRGB(155,155,163), Stroke=Color3.fromRGB(0,0,0),
        SoftButton=Color3.fromRGB(52,52,58), SoftButtonHover=Color3.fromRGB(66,66,73),
        SoftAccent=Color3.fromRGB(44,44,50), SoftAccentHover=Color3.fromRGB(56,56,63),
        ToggleOn=Color3.fromRGB(195,195,205), ToggleOff=Color3.fromRGB(42,42,47), ToggleOff2=Color3.fromRGB(42,42,47),
        ToggleKnobOn=Color3.fromRGB(22,22,24), ToggleKnobOff=Color3.fromRGB(105,105,113),
        InputBg=Color3.fromRGB(33,33,37), SliderBg=Color3.fromRGB(48,48,53), SliderFill=Color3.fromRGB(195,195,205),
        BlacklistHover=Color3.fromRGB(65,38,44), BlacklistLeave=Color3.fromRGB(40,30,34),
    },
    Nexus = {
        Background=Color3.fromRGB(28,28,30), MainBackground=Color3.fromRGB(22,22,24),
        SidebarBg=Color3.fromRGB(16,16,18), ContentBg=Color3.fromRGB(30,30,33),
        SidebarActive=Color3.fromRGB(55,55,60), SidebarActiveText=Color3.fromRGB(255,255,255),
        SidebarInactiveText=Color3.fromRGB(185,185,192),
        Panel=Color3.fromRGB(36,36,40), Row=Color3.fromRGB(44,44,48), RowHover=Color3.fromRGB(56,56,62),
        Accent=Color3.fromRGB(175,175,185), AccentLight=Color3.fromRGB(95,95,105),
        Green=Color3.fromRGB(175,175,185), Red=Color3.fromRGB(210,80,90), Red2=Color3.fromRGB(185,60,72),
        Text=Color3.fromRGB(255,255,255), Dim=Color3.fromRGB(155,155,163), Stroke=Color3.fromRGB(0,0,0),
        SoftButton=Color3.fromRGB(52,52,58), SoftButtonHover=Color3.fromRGB(66,66,73),
        SoftAccent=Color3.fromRGB(44,44,50), SoftAccentHover=Color3.fromRGB(56,56,63),
        ToggleOn=Color3.fromRGB(195,195,205), ToggleOff=Color3.fromRGB(42,42,47), ToggleOff2=Color3.fromRGB(42,42,47),
        ToggleKnobOn=Color3.fromRGB(22,22,24), ToggleKnobOff=Color3.fromRGB(105,105,113),
        InputBg=Color3.fromRGB(33,33,37), SliderBg=Color3.fromRGB(48,48,53), SliderFill=Color3.fromRGB(195,195,205),
        BlacklistHover=Color3.fromRGB(65,38,44), BlacklistLeave=Color3.fromRGB(40,30,34),
    }
}
Theme = {}
for k, v in pairs(Themes.Nexus) do
    Theme[k] = v
end

UIFont = {
    Brand = Enum.Font.GothamBold,
    Title = Enum.Font.GothamBlack,
    Body = Enum.Font.GothamMedium,
    Label = Enum.Font.GothamSemibold,
    Button = Enum.Font.GothamBold,
    Small = Enum.Font.Gotham,
}

UITransparency = {
    MainPanel = 0.04,
    Sidebar = 0.02,
    Content = 0.05,
    Panel = 0.06,
    BottomBar = 0.08,
    Button = 0.08,
    MainButton = 0.10,
    Row = 0.14,
    RowAlt = 0.20,
    SoftRow = 0.12,
    Input = 0.10,
    Toggle = 0,
    Hud = 0.20,
    Alert = 0.24,
    Progress = 0.20,
    ConfigPanel = 0.06,
    SidebarTab = 0.12,
    TargetRow = 0.16,
    TargetRowAlt = 0.30,
    Dropdown = 0.04,
    Outline = 0.72,
    Avatar = 0.18,
}

function applyTheme(themeName)
    local fromTheme = {}
    for k, v in pairs(Theme) do
        fromTheme[k] = v
    end
    
    local toTheme = Themes[themeName] or Themes.Light
    for k, v in pairs(toTheme) do
        Theme[k] = v
    end
    
    -- Helper 1: Explicitly style panels to avoid color collision bugs
    local function updatePanelTheme(f, isMain)
        if not f then return end
        f.BackgroundColor3 = isMain and toTheme.MainBackground or toTheme.Background
        for _, child in ipairs(f:GetChildren()) do
            if child:IsA("UIStroke") then
                child.Color = toTheme.Stroke
            elseif child:IsA("Frame") then
                if child.Size == UDim2.new(1,-24,0,1) then
                    child.BackgroundColor3 = toTheme.AccentLight
                elseif child.BackgroundTransparency == 1 and child.Size == UDim2.new(1,0,0,42) then
                    for _, sub in ipairs(child:GetChildren()) do
                        if sub:IsA("TextLabel") then
                            if sub.TextSize == 16 or sub.TextSize == 12 then
                                sub.TextColor3 = toTheme.Text
                            elseif sub.TextSize == 10 then
                                sub.TextColor3 = toTheme.Dim
                            end
                        end
                    end
                end
            end
        end
    end

    -- Helper 2: Explicitly style bottom bar + sidebar
    local function updateBottomBarTheme()
        if not bottomBar then return end
        bottomBar.BackgroundColor3 = toTheme.Background
        for _, child in ipairs(bottomBar:GetChildren()) do
            if child:IsA("UIStroke") then
                child.Color = toTheme.Stroke
            elseif child:IsA("TextLabel") then
                if child.Name == "BrandTitle" then
                    child.TextColor3 = toTheme.Text
                elseif child.Name == "BrandSep" or child.Name == "BrandDiscord" then
                    child.TextColor3 = toTheme.Dim
                end
            elseif child:IsA("Frame") then
                if child.Name == "BottomBarDiv" or child.Size == UDim2.new(0,1,0,36) or child.Size == UDim2.new(0,1,0,28) then
                    child.BackgroundColor3 = toTheme.Stroke
                end
            end
        end
        local topInfo = bottomBar:FindFirstChild("TopInfo")
        if topInfo then
            for _, child in ipairs(topInfo:GetChildren()) do
                if child:IsA("TextLabel") then
                    if child.Name == "BrandTitle" then child.TextColor3 = toTheme.Text
                    elseif child.Name == "BrandSep" or child.Name == "BrandDiscord" then child.TextColor3 = toTheme.Dim end
                end
            end
        end
        if fpsText then fpsText.TextColor3 = toTheme.Dim end
        if main then
            local sidebar = main:FindFirstChild("Sidebar")
            if sidebar then sidebar.BackgroundColor3 = toTheme.SidebarBg or Color3.fromRGB(0,0,0) end
            local contentBg = main:FindFirstChild("ContentBg")
            if contentBg then
                contentBg.BackgroundColor3 = toTheme.ContentBg or toTheme.MainBackground
            end
        end
        if sidebarProfile then
            for _, child in ipairs(sidebarProfile:GetChildren()) do
                if child:IsA("TextLabel") then
                    if child.Name == "ProfileName" then child.TextColor3 = toTheme.Text
                    elseif child.Name == "ProfileHandle" then child.TextColor3 = toTheme.Dim end
                end
            end
        end
        if _G.styleSidebarTabs and UI and UI.CurrentTab then pcall(_G.styleSidebarTabs, UI.CurrentTab) end
    end

    -- Helper 3: Explicitly style admin panel rows
    local function updateAdminPanelTheme()
        if not apBG then return end
        apBG.BackgroundTransparency = 1
        for uid, row in pairs(apRows) do
            if row and row.Parent then
                row.BackgroundTransparency = 1
                for _, child in ipairs(row:GetChildren()) do
                    if child:IsA("Frame") then
                        if child.Size == UDim2.fromOffset(34,34) then
                            child.BackgroundTransparency = 1
                            local stroke = child:FindFirstChildOfClass("UIStroke")
                            if stroke then stroke.Transparency = 1 end
                        elseif child.ZIndex == 12 then
                            for _, btn in ipairs(child:GetChildren()) do
                                if btn:IsA("TextButton") then
                                    btn.BackgroundTransparency = 1
                                end
                            end
                        end
                    elseif child:IsA("TextLabel") then
                        if child.TextSize == 14 then
                            child.TextColor3 = toTheme.Text
                        elseif child.TextSize == 10 then
                            child.TextColor3 = toTheme.Dim
                        elseif child.TextSize == 11 then
                            child.TextColor3 = toTheme.AccentLight
                        end
                    end
                end
            end
        end
    end

    -- Run explicit styling on all static panels
    pcall(function()
        updatePanelTheme(main, true)
        updateBottomBarTheme()
        updateAdminPanelTheme()
        for _, name in ipairs({"Invisible Steal Panel", "Admin Command Panel", "Command Cooldowns", "Actions", "Steal Panel", "Steal Target"}) do
            updatePanelTheme(panels[name], false)
        end
        updatePanelTheme(actionSettingsPanel, false)
        updatePanelTheme(tpSpeedSettingsPanel, false)
    end)
    
    -- Fallback/Recursive styling for other dynamic frames (excluding mainBody descendants!)
    local function updateInstanceColors(inst)
        local shouldStyle = true
        if inst:IsA("GuiObject") or inst:IsA("UIStroke") then
            -- AVOID COLOR COLLISION BUG: Skip styling anything inside the tab mainBody dynamically
            if mainBody and inst:IsDescendantOf(mainBody) then return end
            
            -- Avoid updating main structures explicitly styled above
            if inst == main or inst == bottomBar or inst == apBG then
                shouldStyle = false
            end
            for _, p in ipairs({"Invisible Steal Panel", "Admin Command Panel", "Command Cooldowns", "Actions", "Steal Panel", "Steal Target"}) do
                if inst == panels[p] then
                    shouldStyle = false
                end
            end
            if inst == actionSettingsPanel or inst == tpSpeedSettingsPanel then
                shouldStyle = false
            end

            if shouldStyle then
                local bgKeys = {
                    Background=true, MainBackground=true, SidebarBg=true, ContentBg=true, SidebarActive=true,
                    Panel=true, Row=true, RowHover=true,
                    SoftButton=true, SoftButtonHover=true, SoftAccent=true, SoftAccentHover=true,
                    ToggleOn=true, ToggleOff=true, ToggleOff2=true, InputBg=true, SliderBg=true,
                    BlacklistHover=true, BlacklistLeave=true
                }
                local textKeys = {
                    Text=true, Dim=true, Accent=true, AccentLight=true, SidebarActiveText=true, SidebarInactiveText=true,
                    Green=true, Red=true, Red2=true, Stroke=true,
                    ToggleKnobOn=true, ToggleKnobOff=true
                }
                local properties = {
                    BackgroundColor3 = bgKeys,
                    TextColor3 = textKeys,
                    PlaceholderColor3 = textKeys,
                    Color = textKeys
                }
                
                for prop, allowedKeys in pairs(properties) do
                    pcall(function()
                        local current = inst[prop]
                        if typeof(current) == "Color3" then
                            -- PROTECT WHITE TEXT ON SPECIFIC BUTTONS FROM THEME CONVERSION
                            if prop == "TextColor3" and inst.Name == "WhiteTextBtn" then
                                return
                            end
                            if prop == "BackgroundColor3" and inst.Name == "WhiteSliderKnob" then
                                return
                            end
                            for k, val in pairs(fromTheme) do
                                if allowedKeys[k] then
                                    if (current.R - val.R)^2 + (current.G - val.G)^2 + (current.B - val.B)^2 < 0.0001 then
                                        inst[prop] = toTheme[k]
                                        break
                                    end
                                end
                            end
                        end
                    end)
                end
            end
        end
        for _, child in ipairs(inst:GetChildren()) do
            updateInstanceColors(child)
        end
    end
    
    pcall(function()
        local sg = playerGui:FindFirstChild("MynxxHub_V3")
        if sg then updateInstanceColors(sg) end
        if _G.updateLogoImage then
            _G.updateLogoImage(themeName ~= "Light")
        end
    end)
    pcall(function()
        local ExploitGui = (gethui and gethui()) or game:GetService("CoreGui")
        local sg = ExploitGui:FindFirstChild("XiPriorityAlertTest")
        if sg then updateInstanceColors(sg) end
    end)
    for _, name in ipairs({"Mynxx_RemoteSell", "Mynxx_StealProgressBar", "XiAdminPanel"}) do
        pcall(function()
            local otherSg = playerGui:FindFirstChild(name)
            if otherSg then updateInstanceColors(otherSg) end
        end)
    end
    
    -- Force specific buttons that must always have white text to keep Color3.new(1,1,1)
    local function forceWhiteText(inst)
        if inst:IsA("TextButton") then
            local txt = inst.Text
            if txt == "ON" or txt == "OFF" or txt == "ADD" or txt == "X" or txt == "▲" or txt == "▼" then
                if inst.Name == "SyncStateBtn" then
                    inst.TextColor3 = Color3.new(1, 1, 1)
                else
                    inst.TextColor3 = Color3.new(1, 1, 1)
                end
            elseif inst.Size == UDim2.new(0, 50, 0, 20) then -- Keybind bind button
                inst.TextColor3 = Color3.new(1, 1, 1)
            end
        elseif inst:IsA("Frame") and inst.Name == "WhiteSliderKnob" then
            inst.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
        end
        for _, child in ipairs(inst:GetChildren()) do
            forceWhiteText(child)
        end
    end
    
    pcall(function()
        local sg = playerGui:FindFirstChild("MynxxHub_V3")
        if sg then forceWhiteText(sg) end
    end)
    
    if Config then
        Config.DarkMode = (themeName == "Dark")
        if saveConfig then saveConfig() end
    end
    pcall(function() if rebuildActions then rebuildActions() end end)
    pcall(function() if rebuildActionSettings then rebuildActionSettings() end end)
    pcall(function() if rebuildTpSpeedSettings then rebuildTpSpeedSettings() end end)
    pcall(function() if loadTab then loadTab(UI.CurrentTab) end end)
end
_G.applyTheme = applyTheme

UI = {Locked=false, OpenMenuKey=Enum.KeyCode.LeftControl, CurrentTab="Auto TP"}

Keybinds = {
    Kick="Y",Clone="F",["Manual TP"]="T",["Invisible Steal"]="U",
    Proximity="P",["Carpet Boost"]="Q",["Open Menu"]="LeftControl",
    ["Ragdoll Self"]="R",["Drop Brainrot"]="NONE",Float="Z",Reset="X",["Auto Buy"]="K",
    ["Click to AP"]="NONE",
    ["WalkSpeed"]="NONE",
}

priorityList = {
    "Headless Horseman",
    "Signore Carapace",
    "Arcadragon",
    "Elefanto Frigo",
    "Strawberry Elephant",
    "John Pork",
    "Meowl",
    "Love Love Bear",
    "Antonio",
    "Skibidi Toilet",
    "Dragon Aquanini",
    "Griffin",
    "Pancake and Syrup",
    "Dragon Gingerini",
    "Fishino Clownino",
    "Tirilikalika Tirilikalako",
    "La Supreme Combinasion",
    "Kalika Bros",
    "Ginger Gerat",
    "Digi Narwhal",
    "Kraken",
    "Venuspino",
    "Bearito Cabinito",
    "Rico Dinero",
    "Globa Steppa",
    "Hydra Bunny",
    "Dug dug dug",
    "Hydra Dragon Cannelloni",
    "Dragon Cannelloni",
    "Jelly Moby",
    "Bunny and Eggy",
    "Ketupat Bros",
    "La Casa Boo",
    "Duggy Bros",
    "Rosey and Teddy",
    "Sammyni Cakini",
    "Guest 666",
    "Gorillo Subwoofero",
    "Los Hackers",
    "Pineaplino",
    "Lazy Ducky",
    "Steakini Fattini",
    "Foxini Lanternini",
    "Rhino Helicopterino",
    "Fragola La La La",
    "Cerberus",
    "Money Money Reindeer",
    "Centrucci Nuclucci",
    "Coco and Mango",
    "Reinito Sleighito",
    "Fragrama and Chocrama",
    "Fortunu and Cashuru",
    "Cooki and Milki",
    "Rubrikiko",
    "Los Chillis",
    "Spooky and Pumpky",
    "Festive 67",
    "Boppin Bunny",
    "Gym Bros",
    "Tralaledon",
    "Jolly Jolly Sahur",
    "Clickerino Crabo",
    "La Food Combinasion",
    "Los Sekolahs",
    "Los Amigos",
    "Capitano Moby",
    "Popcuru and Fizzuru",
    "Cash or Card",
    "Los Spaghettis",
    "Anpali Babel",
    "Sammyni Fattini",
    "Money Money Bros",
    "Caramello Filtrello",
    "John Doe",
    "Los Primos",
    "Los Fruits",
    "Celularcini Viciosini",
    "Los Hotspotsitos",
    "Los Tacoritas",
    "Celestial Pegasus",
    "Sand Sand Sand",
    "Los Mariachis",
    "Tob Tobi Tobi",
    "Ganganzelli Trulala",
    "Malame Amarele",
    "Cloverat Clapat",
    "Burguro And Fryuro",
    "La Anniversary Grande",
    "Brutto Gialutto",
    "Hopilikalika Hopilikalako",
    "Garama and Madundung",
    "Caylusaurus",
    "Swag Soda",
    "La Secret Combinasion",
    "Tukanno Bananno",
    "Gobblino Uniciclino",
    "Swaggy Bros",
    "Nacho Spyder",
    "Quackini Snackini",
    "La Easter Grande",
    "La Romantic Grande",
    "La Lucky Grande",
    "La Jolly Grande",
    "La Taco Combinasion",
    "La Summer Grande",
    "Mangolini Parrocini",
    "Ketchuru and Musturu",
    "Chillin Chili",
    "Orcaledon",
    "W or L",
    "Tuff Toucan",
    "Rosetti Tualetti",
    "La Ginger Sekolah",
    "Tictac Sahur",
    "Ventoliero Pavonero",
    "Los Puggies",
    "Lavadorito Spinito",
    "Spaghetti Tualetti",
    "Gold Gold Gold",
    "Ketupat Kepat",
    "Tang Tang Keletang",
    "Abyssaloco",
    "Lovin Rose",
    "Eviledon",
    "Girafini Raftini",
    "Esok Sekolah",
    "Los Cupids",
    "Mieteteira Bicicleteira",
    "Chipso and Queso",
    "La Spooky Grande",
    "Los Bros",
    "La Extinct Grande",
    "Quesadillo Vampiro",
    "Los 67",
    "Los Mobilis",
    "Churrito Bunnito",
    "Tacorita Bicicleta",
    "Money Money Puggy",
    "Los Jolly Combinasionas",
    "Los Spooky Combinasionas",
    "Los Planitos",
    "Las Sis",
    "Los Sweethearts",
    "Nuclearo Dinossauro",
    "Tacorillo Crocodillo",
    "Pirulitoita Bicicleteira",
    "Telemorte",
    "Nooo My Hotspot",
    "Vulturino Skeletono",
    "Agarrini la Palini",
    "Donkeyturbo Express",
    "Los Candies",
    "Ginger Cisterna",
    "Piccionetta Macchina",
    "Buho de Noelo",
    "Money Money Man",
    "La Sahur Combinasion",
}

actionConfig = {
    ["Ragdoll Self (R)"]=true,["Rejoin PS"]=true,["Rejoin"]=true,["Self Kick"]=true,
    ["Kick (Y)"]=true,["Kick To Private"]=true,["Reset (X)"]=true,
    ["Anti Ragdoll"]=false,["Infinite Jump"]=false,["Carpet Speed"]=false,
}

-- CONFIG (merged)
CONFIG_FILE = "SxeHubConfigs.json"
PS_CODE_FILE = "SxeHubPS.txt"
PrivateServerCode = ""

function loadPSCode()
    pcall(function() if typeof(readfile)=="function" and typeof(isfile)=="function" and isfile(PS_CODE_FILE) then PrivateServerCode = readfile(PS_CODE_FILE) end end)
end
function savePSCode()
    pcall(function() if typeof(writefile)=="function" then writefile(PS_CODE_FILE, PrivateServerCode or "") end end)
end
loadPSCode()

Config = {
    positions={},keybinds={},actions={},locked=false,
    DarkMode=false,
    AntiRagdoll=false,InfiniteJump=false,Float=false,
    DarkTheme=false,DarkThemeIntensity=50,
    AutoResetBalloon=false,AutoKickOnSteal=false,KickToPrivateServer=false,CleanErrorGUIs=false,Steal3DViewer=true,
    LineToBase=false,LineToBrainrot=false,InvisStealAngle=180,SinkSliderValue=8,
    AutoRecoverLagback=true,
    WalkSpeedEnabled=false, WalkSpeedValue=28,
    AutoTPPriority=true, AutoTPHighestGen=false, AutoTPHighestValue=false, FPSBoost=false, FPSBoostUltra=false, XRay=false, FOV=70, KashiOptimizer=false,
    BrainrotESP=true, TimerESP=false, SubspaceMineESP=false, PlayerESP=true, BaseOwnerESP=false, TpTrail=true,
    BrainrotAlert=true,
    AutoBuyEnabled=false, AutoBuyRange=17, AutoGrabSpeed=17, AutoBuyKey="K",
    AutoDestroyTurrets=false, AutoUnlockOnSteal=false,
    AutoInvisDuringSteal=false,
    ClickToAP=false, ClickToAPSingleCommand=false,
    ClickToAPRadius=8,
    SpamBaseOwnerCommands={balloon=true, inverse=true, jail=true, jumpscare=true, morph=true, nightvision=true, ragdoll=true, rocket=true, tiny=true},
    SpamBaseOwnerOrder={"balloon", "inverse", "jail", "jumpscare", "morph", "nightvision", "ragdoll", "rocket", "tiny"},
    SpamBaseOwnerSingleCommand=false,
    ProximityAP=false, ShowJobJoiner=true, AntiBeeDisco=false,
    RemoteSellEnabled=false, AdminPanelUI=true,
    StealHighest=true, StealPriority=false, StealNearest=false,
    AutoStealEnabled=true,
    Unwalk=false,
    Visibilities = {
        ["Invisible Steal Panel"] = true,
        ["Admin Command Panel"] = true,
        ["Command Cooldowns"] = true,
        ["Actions"] = true,
        ["Steal Panel"] = true,
        ["Steal Target"] = true,
    },
    TpSettings = {
        Tool="Flying Carpet", TpKey="T", CloneKey="V", CarpetSpeedKey="Q",
        InfiniteJump=false, DelayVal=0.4, CloneDelayVal=0.1,
        RagdollTP=false, FPSWait=false, FlyTPSpeed=160, FlyTPCloseSpeed=75,
        GrabbleTP=false, GrabbleTPSpeed=450,
        TpOnLoad=false, MinGenForTp="", MinGenForGrab="",
        BrainrotCarpet=false,
    },
    PriorityList=priorityList,
    RemovedFromPriority={},
}

-- Brainrot ESP
brainrotESPEnabled=Config.BrainrotESP; local brainrotBillboards={}
function createBrainrotBillboard(data)
    local bb=Instance.new("BillboardGui"); bb.Name="BrainrotESP_"..data.uid; bb.Size=UDim2.new(0,160,0,44)
    bb.StudsOffset=Vector3.new(0,1.8,0); bb.AlwaysOnTop=true; bb.LightInfluence=0; bb.MaxDistance=3000
    local container=Instance.new("Frame",bb); container.Size=UDim2.new(1,0,1,0); container.BackgroundColor3=Color3.new(0,0,0)
    container.BackgroundTransparency=0.6; container.BorderSizePixel=0; Instance.new("UICorner",container).CornerRadius=UDim.new(0,4)
    local stroke=Instance.new("UIStroke",container); stroke.Color=Color3.new(1,1,1); stroke.Thickness=1.2; stroke.Transparency=0.2
    local nameLabel=Instance.new("TextLabel",container); nameLabel.Size=UDim2.new(1,-6,0.55,0); nameLabel.Position=UDim2.new(0,3,0,1)
    nameLabel.BackgroundTransparency=1; nameLabel.Font=Enum.Font.GothamBold; nameLabel.TextSize=13;     nameLabel.TextColor3=Color3.fromRGB(255,255,255)
    nameLabel.TextStrokeTransparency=0; nameLabel.TextStrokeColor3=Color3.new(0,0,0); nameLabel.Text=data.name or "???"; nameLabel.TextXAlignment=Enum.TextXAlignment.Center
    local genLabel=Instance.new("TextLabel",container); genLabel.Size=UDim2.new(1,-6,0.45,0); genLabel.Position=UDim2.new(0,3,0.55,0)
    genLabel.BackgroundTransparency=1; genLabel.Font=Enum.Font.GothamBold; genLabel.TextSize=11;     genLabel.TextColor3=Color3.fromRGB(255,220,50)
    genLabel.TextStrokeTransparency=0; genLabel.TextStrokeColor3=Color3.new(0,0,0)
    local gt=data.genText or ""; local mut=data.mutation and data.mutation~="None" and (" ["..data.mutation.."]") or ""
    genLabel.Text=gt..mut; genLabel.TextXAlignment=Enum.TextXAlignment.Center
    return bb
end
function refreshBrainrotESP()
    if not brainrotESPEnabled then return end; if _G._isTpMoving then return end; local cache=SharedState.AllAnimalsCache; if not cache or #cache==0 then return end
    local maxGV=0; for _,d in ipairs(cache) do if d.genValue>maxGV then maxGV=d.genValue end end
    local hasTop=false; for _,d in ipairs(cache) do if d.genValue>=10000000 then hasTop=true; break end end
    local threshold=hasTop and 10000000 or maxGV*0.5
    local seen={}
    for _,data in ipairs(cache) do if data.genValue>=threshold then seen[data.uid]=true
        if not brainrotBillboards[data.uid] then
            local adornee=findAdorneeGlobal(data)
            if adornee then local bb=createBrainrotBillboard(data); bb.Adornee=adornee; bb.Parent=adornee
                brainrotBillboards[data.uid]={bb=bb}
            end
        end
    end end
    for uid,entry in pairs(brainrotBillboards) do if not seen[uid] then
        if entry.bb then entry.bb:Destroy() end; brainrotBillboards[uid]=nil
    end end
end
function clearBrainrotESP() for _,e in pairs(brainrotBillboards) do if e.bb then e.bb:Destroy() end end; brainrotBillboards={} end
task.spawn(function() while true do task.wait(0.3); if brainrotESPEnabled then pcall(refreshBrainrotESP) end end end)

suffixes = {
    k = 1e3,
    m = 1e6,
    b = 1e9,
    t = 1e12,
    q = 1e15,
    qi = 1e18,
    qd = 1e18,
    qn = 1e18,
    sx = 1e21,
    sp = 1e24,
    oc = 1e27,
    no = 1e30,
    dc = 1e33,
    ud = 1e36,
    dd = 1e39,
    td = 1e42,
    qad = 1e45,
    qid = 1e48,
    sxd = 1e51,
    spd = 1e54,
    ocd = 1e57,
    nod = 1e60,
    vg = 1e63,
}

function parseMinGen(str)
    if not str or type(str) ~= "string" then return 0 end
    str = str:gsub("%s", ""):lower():gsub("/s$", "")
    if str == "" then return 0 end
    local numStr, suffix = str:match("^([%d%.]+)(%a*)$")
    if not numStr then return 0 end
    local num = tonumber(numStr)
    if not num or num < 0 then return 0 end
    if suffix ~= "" then
        local mult = suffixes[suffix]
        if mult then
            return num * mult
        end
    end
    return num
end

function canUseFiles() return typeof(readfile)=="function" and typeof(writefile)=="function" and typeof(isfile)=="function" end
loadConfig = function()
    if not canUseFiles() then return end
    local ok,data=pcall(function() if isfile(CONFIG_FILE) then return HttpService:JSONDecode(readfile(CONFIG_FILE)) end end)
    if ok and type(data)=="table" then
        for k,v in pairs(data) do
            if k == "PriorityList" or k == "RemovedFromPriority" then
                Config[k] = v
            elseif type(v) == "table" and type(Config[k]) == "table" then
                for subk, subv in pairs(v) do
                    Config[k][subk] = subv
                end
            else
                Config[k] = v
            end
        end
        if type(Config.positions)~="table" then Config.positions={} end
        if type(Config.keybinds)~="table" then Config.keybinds={} end
        if type(Config.actions)~="table" then Config.actions={} end
        if type(Config.RemovedFromPriority)~="table" then Config.RemovedFromPriority={} end
        local removedSet = {}
        for _, rn in ipairs(Config.RemovedFromPriority) do removedSet[rn] = true end
        if type(Config.PriorityList)=="table" then
            if #Config.PriorityList == 0 then
                Config.PriorityList = priorityList
            else
                local present = {}
                for _, name in ipairs(Config.PriorityList) do
                    present[name] = true
                end
                for _, name in ipairs(priorityList) do
                    if not present[name] and not removedSet[name] then
                        table.insert(Config.PriorityList, name)
                    end
                end
                priorityList = Config.PriorityList
            end
        else
            Config.PriorityList = priorityList
        end
        -- Deduplicate priority list
        if type(Config.PriorityList) == "table" then
            local seen = {}
            local cleanList = {}
            for _, name in ipairs(Config.PriorityList) do
                if not seen[name] then
                    seen[name] = true
                    table.insert(cleanList, name)
                end
            end
            Config.PriorityList = cleanList
            priorityList = cleanList
        end
    end
end
saveConfig = function()
    if not canUseFiles() then return end
    task.spawn(function() pcall(function() writefile(CONFIG_FILE, HttpService:JSONEncode(Config)) end) end)
end

-- BASE64 ENCODER/DECODER FOR CONFIG SHARING (Wrapped in do-scope to save global registers)
do
    local b64chars = 'ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789+/'
    local function base64Encode(data)
        local len = #data
        local t = {}
        for i = 1, len, 3 do
            local a = data:byte(i)
            local b = data:byte(i + 1) or 0
            local c = data:byte(i + 2) or 0
            local n = a * 65536 + b * 256 + c
            t[#t + 1] = b64chars:sub(math.floor(n / 262144) + 1, math.floor(n / 262144) + 1)
            t[#t + 1] = b64chars:sub(math.floor(n / 4096) % 64 + 1, math.floor(n / 4096) % 64 + 1)
            t[#t + 1] = (i + 1 <= len) and b64chars:sub(math.floor(n / 64) % 64 + 1, math.floor(n / 64) % 64 + 1) or "="
            t[#t + 1] = (i + 2 <= len) and b64chars:sub(n % 64 + 1, n % 64 + 1) or "="
        end
        return table.concat(t)
    end

    local function base64Decode(data)
        data = data:gsub('[^' .. b64chars .. '=]', '')
        local len = #data
        local t = {}
        local lookup = {}
        for i = 1, 64 do lookup[b64chars:sub(i, i)] = i - 1 end
        local i = 1
        while i <= len do
            local c1 = lookup[data:sub(i, i)]
            local c2 = lookup[data:sub(i + 1, i + 1)]
            local s3 = data:sub(i + 2, i + 2)
            local s4 = data:sub(i + 3, i + 3)
            local c3 = lookup[s3]
            local c4 = lookup[s4]
            if not c1 or not c2 then break end
            local n = c1 * 262144 + c2 * 4096 + (c3 or 0) * 64 + (c4 or 0)
            t[#t + 1] = string.char(math.floor(n / 65536))
            if s3 ~= "=" then
                t[#t + 1] = string.char(math.floor(n / 256) % 256)
            end
            if s4 ~= "=" then
                t[#t + 1] = string.char(n % 256)
            end
            i = i + 4
        end
        return table.concat(t)
    end

    _G.importConfig = function(str)
        if not str or str == "" then
            ShowNotification("IMPORT ERROR", "Config string is empty")
            return false
        end
        
        local success, parsed = pcall(function()
            return HttpService:JSONDecode(str)
        end)
        
        if not success or type(parsed) ~= "table" then
            local decoded
            pcall(function() decoded = base64Decode(str:gsub("%s", "")) end)
            if decoded then
                success, parsed = pcall(function() return HttpService:JSONDecode(decoded) end)
            end
        end

        if not success or type(parsed) ~= "table" then
            ShowNotification("IMPORT ERROR", "Invalid config data")
            return false
        end

        for k, v in pairs(parsed) do
            Config[k] = v
        end
        
        if type(Config.PriorityList) == "table" then priorityList = Config.PriorityList end
        
        saveConfig()

        for k, v in pairs(Config.keybinds or {}) do
            if Keybinds[k] ~= nil and type(v) == "string" then
                Keybinds[k] = v
                if k == "Open Menu" and Enum.KeyCode[v] then
                    UI.OpenMenuKey = Enum.KeyCode[v]
                end
            end
        end

        for k, v in pairs(Config.actions or {}) do
            if actionConfig[k] ~= nil then
                actionConfig[k] = v and true or false
            end
        end

        if type(Config.locked) == "boolean" then
            UI.Locked = Config.locked
            if _G.updateResizeGrips then pcall(_G.updateResizeGrips) end
        end

        initToggles()

        if setFPSBoost then pcall(setFPSBoost, Config.FPSBoost) end
        if setFPSBoostUltra then pcall(setFPSBoostUltra, Config.FPSBoostUltra) end
        -- Unlock FPS is ALWAYS on by default, whatever the saved config says
        if setXRay then pcall(setXRay, Config.XRay) end
        if setInfiniteJump then pcall(setInfiniteJump, Config.InfiniteJump) end
        if setCarpetSpeed then pcall(setCarpetSpeed, Config["Carpet Speed"] or Config.CarpetSpeed or false) end
        -- ProximityAP always starts OFF, never loaded from config
    Config.ProximityAP = false
        -- Auto Buy always starts OFF on a new server (never restored)
        Config.AutoBuyEnabled = false
        if setStealMode then
            if Config.StealHighest then pcall(setStealMode, "Highest")
            elseif Config.StealPriority then pcall(setStealMode, "Priority")
            elseif Config.StealNearest then pcall(setStealMode, "Nearest")
            end
        end
        if updateMovementPanelLabels then pcall(updateMovementPanelLabels) end

        rebuildActions()
        rebuildActionSettings()
        loadTab(UI.CurrentTab)

        ShowNotification("CONFIG SYSTEM", "Config imported successfully!")
        return true
    end

    _G.exportConfig = function()
        local ok, str = pcall(function()
            return HttpService:JSONEncode(Config)
        end)
        if not ok or not str then
            ShowNotification("EXPORT ERROR", "Failed to encode config")
            return nil
        end
        local cbSuccess = false
        if typeof(setclipboard) == "function" then
            pcall(function() setclipboard(str); cbSuccess = true end)
        elseif typeof(toclipboard) == "function" then
            pcall(function() toclipboard(str); cbSuccess = true end)
        end
        if cbSuccess then
            ShowNotification("CONFIG SYSTEM", "Config copied to clipboard!")
        else
            ShowNotification("CONFIG SYSTEM", "Config generated! Copy from share box.")
        end
        return str
    end
end
function serializePos(pos) return {xs=pos.X.Scale,xo=pos.X.Offset,ys=pos.Y.Scale,yo=pos.Y.Offset} end
function rememberPosition(name, frame) if not name or not frame then return end; Config.positions[name]=serializePos(frame.Position); saveConfig() end
function applySavedPosition(name, frame)
    if not name or not frame then return end; local d=Config.positions and Config.positions[name]
    if d then frame.Position=UDim2.new(d.xs or 0,d.xo or 0,d.ys or 0,d.yo or 0) end
end
function initToggles()
    setToggle("Anti Ragdoll", Config.AntiRagdoll, true)
    setToggle("Auto Reset Balloon", Config.AutoResetBalloon, true)
    setToggle("Infinite Jump", Config.InfiniteJump, true)
    setToggle("Auto Kick", Config.AutoKickOnSteal, true)
        setToggle("Auto Buy", false, true)
    setToggle("Auto Steal", Config.AutoStealEnabled, true)
    setToggle("Steal Highest", Config.StealHighest, true)
    setToggle("Steal Priority", Config.StealPriority, true)
    setToggle("Steal Nearest", Config.StealNearest, true)
    setToggle("Click to AP", Config.ClickToAP, true)
    setToggle("ClickToAP", Config.ClickToAP, true)
    setToggle("Click AP Single Cmd", Config.ClickToAPSingleCommand, true)
    setToggle("ClickToAPSingle", Config.ClickToAPSingleCommand, true)
    setToggle("FPS Boost (normal)", Config.FPSBoost, true)
    setToggle("FPS Boost (normal)", Config.FPSBoost, true)
    setToggle("FPS Boost Ultra", Config.FPSBoostUltra, true)
    setToggle("FPSBoostUltra", Config.FPSBoostUltra, true)
    setToggle("XRay", Config.XRay, true)
    setToggle("X-Ray", Config.XRay, true)
    setToggle("Xray", Config.XRay, true)
    setToggle("Proximity", Config.ProximityAP, true)

    setToggle("Player ESP", Config.PlayerESP, true)
    setToggle("Brainrot ESP", Config.BrainrotESP, true)
    setToggle("Timer ESP", Config.TimerESP, true)
    setToggle("Subspace Mine ESP", Config.SubspaceMineESP, true)
    setToggle("Base Owner ESP", Config.BaseOwnerESP, true)
    setToggle("Auto Destroy Turrets", Config.AutoDestroyTurrets, true)
    setToggle("Anti-Bee & Anti-Disco", Config.AntiBeeDisco, true)
    setToggle("AntiBeeDisco", Config.AntiBeeDisco, true)
    setToggle("Admin Panel UI", Config.AdminPanelUI, true)
    setToggle("Auto Invis During Steal", Config.AutoInvisDuringSteal, true)
    setToggle("Auto TP Priority Mode", Config.AutoTPPriority, true)
    setToggle("Auto TP Highest Gen", Config.AutoTPHighestGen, true)
    setToggle("Auto TP Highest Value", Config.AutoTPHighestValue, true)
    setToggle("Unwalk", Config.Unwalk, true)
    setToggle("Stealing ESP", Config.StealingESP, true)
    setToggle("WalkSpeed", Config.WalkSpeedEnabled, true)
    setToggle("Dark Mode", Config.DarkMode, true)
    setToggle("DarkMode", Config.DarkMode, true)
    setToggle("Grabble TP", Config.TpSettings.GrabbleTP or false, true)
    setToggle("KashiOptimizer", Config.KashiOptimizer, true)
end

loadConfig()
-- Nearest = préférence de session seulement : repasse en Priority à chaque nouveau serveur
if Config.StealNearest or Config.StealMode == "Nearest" then
    Config.StealNearest = false
    Config.StealPriority = true
    Config.StealHighest = false
    Config.StealMode = "Priority"
    saveConfig()
end
-- (old DarkMode boot override removed: the Nexus theme is the base now and
-- overriding Theme with Dark before widget creation broke toggle colors)
for k,v in pairs(Config.keybinds or {}) do if Keybinds[k]~=nil and type(v)=="string" then Keybinds[k]=v; if k=="Open Menu" and Enum.KeyCode[v] then UI.OpenMenuKey=Enum.KeyCode[v] end end end
for k,v in pairs(Config.actions or {}) do if actionConfig[k]~=nil then actionConfig[k]=v and true or false end end
if type(Config.locked)=="boolean" then UI.Locked=Config.locked end
initToggles()

-- EARLY BOOT: FPS Boost + Unwalk imediatos (sem esperar 12k+ linhas de UI)
if Config.FPSBoost then
    Config.FPSBoost=true
    Lighting.GlobalShadows=false; Lighting.Brightness=2; Lighting.FogEnd=9e9; Lighting.FogStart=0
    Lighting.EnvironmentDiffuseScale=0; Lighting.EnvironmentSpecularScale=0
    pcall(function() Lighting.ShadowSoftness=0 end)
    for _,v in pairs(Lighting:GetChildren()) do
        if v.Name ~= "MynxxDarkTheme" and (v:IsA("BloomEffect") or v:IsA("BlurEffect") or v:IsA("ColorCorrectionEffect") or v:IsA("SunRaysEffect") or v:IsA("DepthOfFieldEffect")) then pcall(function() v.Enabled=false end) elseif v:IsA("Atmosphere") then pcall(function() v:Destroy() end) end
    end
    local _fpsDescs = Workspace:GetDescendants()
    for i = 1, #_fpsDescs do
        local obj = _fpsDescs[i]
        if _G.__laser and _G.__laser.IsLaserObject and _G.__laser.IsLaserObject(obj) then
            pcall(_G.__laser.UpdateLaserObject, obj)
        elseif not (type(IsProtected)=="function" and IsProtected(obj)) then
            if obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam") or obj:IsA("Smoke") or obj:IsA("Fire") or obj:IsA("Sparkles") then
                if not (obj:IsA("Beam") and (obj.Name == "PlotBeam" or obj.Name == "PlotBeamGlow" or obj.Name == "BestPetBeam")) then pcall(function() obj.Enabled=false end) end
            end
            if obj:IsA("BasePart") then pcall(function() obj.Material=Enum.Material.Plastic; obj.CastShadow=false end) end
            if obj:IsA("SurfaceAppearance") or obj:IsA("Texture") or obj:IsA("Decal") then pcall(function() obj:Destroy() end) end
            if obj:IsA("PointLight") or obj:IsA("SpotLight") or obj:IsA("SurfaceLight") then pcall(function() obj.Enabled=false; obj.Shadows=false end) end
        end
    end
    if not fpsBoostConnection then
        fpsBoostConnection = Workspace.DescendantAdded:Connect(function(obj)
            if not Config.FPSBoost then return end
            if _G.__laser and _G.__laser.IsLaserObject and _G.__laser.IsLaserObject(obj) then pcall(_G.__laser.UpdateLaserObject, obj)
            elseif not (type(IsProtected)=="function" and IsProtected(obj)) then
                if obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam") or obj:IsA("Smoke") then
                    if not (obj:IsA("Beam") and (obj.Name == "PlotBeam" or obj.Name == "PlotBeamGlow" or obj.Name == "BestPetBeam")) then pcall(function() obj.Enabled=false end) end
                end
                if obj:IsA("BasePart") then pcall(function() obj.Material=Enum.Material.Plastic; obj.CastShadow=false end) end
                if obj:IsA("PointLight") or obj:IsA("SpotLight") or obj:IsA("SurfaceLight") then pcall(function() obj.Enabled=false; obj.Shadows=false end) end
            end
        end)
    end
end
if Config.FPSBoostUltra then
    Config.FPSBoostUltra=true
    pcall(function()
        settings().Rendering.QualityLevel=Enum.QualityLevel.Level01
        settings().Rendering.MeshPartDetailLevel=Enum.MeshPartDetailLevel.Level01
        settings().Physics.AllowSleep=true
    end)
    pcall(function() settings().Physics.PhysicsEnvironmentalThrottle=Enum.EnviromentalPhysicsThrottle.Skip end)
    if sethiddenproperty then pcall(function() sethiddenproperty(Lighting,"Technology",Enum.Technology.Compatibility) end) end
end
if Config.Unwalk then
    task.spawn(function()
        local char = player.Character or player.CharacterAdded:Wait()
        local hum = char:WaitForChild("Humanoid", 5)
        if hum then
            local animator = hum:FindFirstChildOfClass("Animator")
            local animate = char:FindFirstChild("Animate")
            if animate then animate.Disabled = true end
            if animator then
                local ok, tracks = pcall(function() return animator:GetPlayingAnimationTracks() end)
                if ok and tracks then
                    for _, t in ipairs(tracks) do pcall(function() t:Stop(0) end) end
                end
            end
        end
    end)
end

do
local _plotCache={}
local _sync
local _realCh
local function _getSync()
if _sync then return _sync end
local ok,s=pcall(function()return require(game:GetService("ReplicatedStorage"):WaitForChild("Packages"):WaitForChild("Synchronizer"))end)
if ok then _sync=s end
return _sync
end
local function _installHook()
local sync=_getSync()
if not sync or type(sync.GetAllChannels)~="function" then return end
local orig=sync.GetAllChannels
sync.GetAllChannels=function(...)
local r=orig(...)
if not _realCh and type(r)=="table" then _realCh=r end
return r
end
end
_installHook()
_G.XenSyncAll=function()return _realCh or _plotCache end
_G.XenSyncGet=function(idx)
if idx==nil then return nil end
if _realCh then
local ch=rawget(_realCh,idx)
if ch and type(ch)=="table" and type(ch.Get)=="function" then return ch end
for k,v in pairs(_realCh)do if tostring(k)==idx and type(v)=="table" and type(v.Get)=="function" then return v end end
end
if _plotCache[idx]then return _plotCache[idx]end
local sync=_getSync()
if not sync or type(sync.GetTableFromChannel)~="function" then return nil end
local ok,ct=pcall(sync.GetTableFromChannel,sync,idx)
if ok and type(ct)=="table" then
local proxy=setmetatable({},{
__index=function(_,k)
if k=="Get"then return function(_,f)return ct[f]end end
if k=="GetTable"then return function()return ct end end
return ct[k]
end
})
_plotCache[idx]=proxy
return proxy
end
return nil
end
_G._xenRawCT=function(plotName)
local c=_G.XenSyncGet(plotName)
if not c then return nil end
if type(c.GetTable)=="function" then local ok,t=pcall(c.GetTable,c);if ok then return t end end
return nil
end
local _AD,_MD,_TD
local function _data()
if _AD then return true end
local ok=pcall(function()
local d=game:GetService("ReplicatedStorage"):WaitForChild("Datas")
_AD=require(d:WaitForChild("Animals"))
_MD=require(d:WaitForChild("Mutations"))
_TD=require(d:WaitForChild("Traits"))
end)
return ok and _AD~=nil
end
_G._xenGen=function(index,mutation,traits)
if not _data() then return 0 end
local info=_AD[index]
if not info or not info.Generation then return 0 end
local mult=1
if mutation and mutation~="None" and mutation~="" then
local m=_MD[mutation]
if m and m.Modifier then mult=mult+m.Modifier end
end
if traits then
if type(traits)=="string"and traits~="None"then
for tr in traits:gmatch("[^, ]+")do
local t=_TD[tr]
if t and t.MultiplierModifier then mult=mult+t.MultiplierModifier end
end
elseif type(traits)=="table"then
for _,tr in ipairs(traits)do
local t=_TD[tr]
if t and t.MultiplierModifier then mult=mult+t.MultiplierModifier end
end
end
end
return info.Generation*mult
end
_G._xenAnimShim=setmetatable({GetGeneration=function(_,index,mutation,traits)return _G._xenGen(index,mutation,traits)end},{
__index=function(_,k)
local ok,real=pcall(function()return require(game:GetService("ReplicatedStorage"):WaitForChild("Shared"):WaitForChild("Animals"))end)
if ok and type(real)=="table" then return rawget(real,k) end
return nil
end})
end
_G.stealthGet=function(n) return _G.XenSyncGet(n) end
_G.SyncInt={_cache={},_data=nil}




do
    local ReplicatedStorage = game:GetService("ReplicatedStorage")
    local netFolder = ReplicatedStorage:WaitForChild("Packages"):WaitForChild("Net")
    local getupvalues = debug.getupvalues or getupvalues
    local getinfo = debug.getinfo or getinfo

    local F1, F2, secret

    local function isGuid(s)
        return type(s) == "string" and #s == 36
            and s:match("^%x%x%x%x%x%x%x%x%-%x%x%x%x%-%x%x%x%x%-%x%x%x%x%-%x%x%x%x%x%x%x%x%x%x%x%x$") ~= nil
    end

    local function build()
        if F1 and secret then return true end
        local jobId = game.JobId
        local a, b
        local cands, seen = {}, {}
        for _, v in getgc(true) do
            if typeof(v) == "function" then
                local ok, info = pcall(getinfo, v)
                if ok and info and info.source == "=ReplicatedStorage.Packages.Net.Net" then
                    local ok2, ups = pcall(getupvalues, v)
                    if ok2 and ups then
                        for _, up in pairs(ups) do
                            if type(up) == "table" then
                                local x, y, z = rawget(up, 1), rawget(up, 2), rawget(up, 3)
                                if not a and type(x) == "function" and type(y) == "function" and type(z) == "string" then
                                    a, b = clonefunction(x), clonefunction(y)
                                end
                                for _, e in pairs(up) do
                                    if isGuid(e) and e ~= jobId and not seen[e] then seen[e] = true; cands[#cands + 1] = e end
                                end
                            elseif isGuid(up) and up ~= jobId and not seen[up] then
                                seen[up] = true; cands[#cands + 1] = up
                            end
                        end
                    end
                end
            end
        end
        if not a then return false end
        F1, F2 = a, b
        local cipher = F2("UseItem", jobId)
        if #cands == 1 then secret = cands[1]; return true end
        for _, sec in ipairs(cands) do
            local ok, h = pcall(function() return F1(buffer.fromstring(cipher), buffer.fromstring(sec .. jobId)) end)
            if ok and type(h) == "string" and netFolder:FindFirstChild("RE/" .. h) then secret = sec; return true end
        end
        return false
    end

    local function resolve(prefix, name)
        if not (F1 and secret) and not build() then return nil end
        local jobId = game.JobId
        local ok, h = pcall(function()
            local c = F2(name, jobId)
            return F1(buffer.fromstring(c), buffer.fromstring(secret .. jobId))
        end)
        if not ok or type(h) ~= "string" then return nil end
        return netFolder:FindFirstChild(prefix .. "/" .. h)
    end

    _G.Net = {
        RemoteEvent           = function(_, name) return resolve("RE", name) end,
        RemoteFunction        = function(_, name) return resolve("RF", name) end,
        UnreliableRemoteEvent = function(_, name) return resolve("URE", name) end,
    }
    _G.XenNet = _G.Net
end

-- Decrypted net
Decrypted=setmetatable({},{__index=function(S,ez)
    if type(ez)~="string" or not _G.Net then return nil end
    local pfx,nm=ez:match("^(R[EF])/(.+)$")
    if not pfx then return nil end
    local ok,r=pcall(function()
        if pfx=="RF" then return _G.Net:RemoteFunction(nm) end
        return _G.Net:RemoteEvent(nm)
    end)
    if ok and r then rawset(Decrypted,ez,r) end
    return ok and r or nil
end})

-- COOLDOWN TRACKER
ACTION_COOLDOWNS = {ragdoll=30,jail=60,rocket=120,balloon=30,inverse=30,jumpscare=30,tiny=30,morph=30,nightvision=30}
lastActionUse = {}

-- Read real timer from the game's AdminPanel UI
function _readRealAdminTimer(cmd)
    local realAdminGui = playerGui:FindFirstChild("AdminPanel")
    if not realAdminGui then return nil end
    local ok, contentScroll = pcall(function() return realAdminGui.AdminPanel.Content.ScrollingFrame end)
    if not ok or not contentScroll then return nil end
    local cmdBtn = contentScroll:FindFirstChild(cmd)
    if not cmdBtn then return nil end
    local timerLabel = cmdBtn:FindFirstChild("Timer")
    if not timerLabel or not timerLabel.Visible then return 0 end
    local num = tonumber(timerLabel.Text:match("%d+"))
    return num or 0
end

function apIsOnCooldown(cmd)
    local realTime = _readRealAdminTimer(cmd)
    if realTime ~= nil then return realTime > 0 end
    local l=lastActionUse[cmd]; local cd=ACTION_COOLDOWNS[cmd] or 0; return l and cd>0 and (tick()-l)<cd
end
function apGetRemaining(cmd)
    local realTime = _readRealAdminTimer(cmd)
    if realTime ~= nil then return realTime end
    local l=lastActionUse[cmd]; local cd=ACTION_COOLDOWNS[cmd] or 0; if not l then return 0 end; return math.max(0,cd-(tick()-l))
end
function apStartCooldown(cmd) lastActionUse[cmd]=tick() end
AP_ALL_COMMANDS={"balloon","inverse","jail","jumpscare","morph","nightvision","ragdoll","rocket","tiny"}
AP_COMMAND_EMOJIS={balloon="🎈",inverse="🔄",jail="🔒",jumpscare="👻",morph="🎭",nightvision="🌙",ragdoll="🤸",rocket="🚀",tiny="🐜"}
if not Config.ClickToAPCommands then
    Config.ClickToAPCommands = {}
    for _, cmd in ipairs(AP_ALL_COMMANDS) do Config.ClickToAPCommands[cmd] = true end
end
if not Config.AdminPanelButtons then
    Config.AdminPanelButtons = {ragdoll=true, jail=true, rocket=true, balloon=true}
end
if not Config.ClickToAPRadius then
    Config.ClickToAPRadius = 8
end
if not Config.SpamBaseOwnerCommands then
    Config.SpamBaseOwnerCommands = {}
    for _, cmd in ipairs(AP_ALL_COMMANDS) do Config.SpamBaseOwnerCommands[cmd] = true end
end
if not Config.SpamBaseOwnerOrder then
    Config.SpamBaseOwnerOrder = {}
    for i, cmd in ipairs(AP_ALL_COMMANDS) do Config.SpamBaseOwnerOrder[i] = cmd end
end
if Config.SpamBaseOwnerSingleCommand == nil then
    Config.SpamBaseOwnerSingleCommand = false
end
setToggle("SpamBaseOwnerSingleCommand", Config.SpamBaseOwnerSingleCommand or false)

_G.apBlacklist = Config.apBlacklist or {}

function isPlayerBlacklisted(plr)
    if not plr then return false end
    local uid = plr.UserId
    return _G.apBlacklist[uid] == true or _G.apBlacklist[tostring(uid)] == true
end

-- Cached Synchronizer module. Re-requiring + WaitForChild on every call (inside
-- per-player loops) was a major admin-panel lag source; cache it once.
do
    local cached
    function _G.__getSync()
        if cached then return cached end
        local ok, mod = pcall(function()
            local pkgs = ReplicatedStorage:FindFirstChild("Packages")
            return pkgs and require(pkgs:WaitForChild("Synchronizer", 5))
        end)
        if ok then cached = mod end
        return cached
    end
end

-- NOTE: current-base-owner detection lives below as _G.__getCurrentBaseOwnerId,
-- right after getPlayerBaseInfo (it depends on getPlotAtPosition/getPlotOwner,
-- which are declared further down). Every player here owns their own base, so a
-- "who owns ANY base" check would (correctly but uselessly) flag everyone.

function getPlotOwner(plot)
    if not plot then return nil end
    local Synchronizer = _G.__getSync()
    if Synchronizer then
        local ch = _G.XenSyncGet(plot.Name)
        if ch then
            local owner = ch:Get("Owner")
            if owner then
                if typeof(owner) == "Instance" and owner:IsA("Player") then
                    return owner
                elseif type(owner) == "table" and owner.Name then
                    return Players:FindFirstChild(owner.Name)
                elseif type(owner) == "number" then
                    return Players:GetPlayerByUserId(owner)
                end
            end
        end
    end
    local sign = plot:FindFirstChild("PlotSign")
    local textLabel = sign
        and sign:FindFirstChild("SurfaceGui")
        and sign.SurfaceGui:FindFirstChild("Frame")
        and sign.SurfaceGui.Frame:FindFirstChild("TextLabel")
    if textLabel then
        local baseText = textLabel.Text
        local nickname = (baseText and baseText:match("^(.-)'")) or baseText
        if nickname then
            for _, p in ipairs(Players:GetPlayers()) do
                if (p.DisplayName == nickname) or (p.Name == nickname) then
                    return p
                end
            end
        end
    end
    return nil
end

function getPlotAtPosition(pos)
    local plots = Workspace:FindFirstChild("Plots")
    if not plots then return nil end
    local closestPlot = nil
    local minDistance = math.huge
    for _, plot in ipairs(plots:GetChildren()) do
        local plotPos
        if plot:IsA("Model") then
            plotPos = plot.PrimaryPart and plot.PrimaryPart.Position or plot:GetPivot().Position
        else
            plotPos = plot.Position
        end
        if plotPos then
            local distH = math.sqrt((pos.X - plotPos.X)^2 + (pos.Z - plotPos.Z)^2)
            if distH < minDistance then
                minDistance = distH
                closestPlot = plot
            end
        end
    end
    if closestPlot and minDistance < 72 then
        return closestPlot
    end
    return nil
end

function getPlayerBaseInfo(plr)
    if not plr or not plr.Character then return nil, nil end
    local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return nil, nil end
    local plot = getPlotAtPosition(hrp.Position)
    if plot then
        return plot, getPlotOwner(plot)
    end
    return nil, nil
end

-- userId of the owner of the base the LOCAL player is currently standing in
-- (nil if none). Memoized ~0.4s and shared by all admin rows, so only ONE row
-- shows "Base Owner": the owner of the base you're actually in.
do
    local cachedId, lastT = nil, 0
    function _G.__getCurrentBaseOwnerId()
        local now = os.clock()
        if (now - lastT) < 0.4 then return cachedId end
        lastT = now
        cachedId = nil
        local _, owner = getPlayerBaseInfo(LocalPlayer)
        if owner then cachedId = owner.UserId end
        return cachedId
    end
end

-- MORPH TRACKING: the AP "morph" command parents a brainrot-named model to the
-- TARGET's character. That model matched the carried-brainrot scan below, so
-- the admin panel / ESP showed the MORPH's name instead of the stolen pet.
-- Fix: any brainrot-named model that is in the character while the player is
-- NOT stealing is a morph (or a held own pet) -- remember those names and
-- exclude them from the carried scan once the steal starts.
_MynxxMorphCache = _MynxxMorphCache or {} -- [userId] = {t=os.clock(), names={set}}
-- Per-steal-session resolved name. locked=true (server attribute / carried
-- model) is trusted for the whole steal; locked=false (podium guess) keeps
-- being re-checked and upgrades as soon as a reliable source resolves it.
_MynxxStealSession = _MynxxStealSession or {} -- [userId] = {name, owner, locked, t}
Players.PlayerRemoving:Connect(function(plr)
    _MynxxMorphCache[plr.UserId] = nil
    _MynxxStealSession[plr.UserId] = nil
end)

function getStealingInfo(plr)
    if not plr or not plr.Character then return nil, nil end
    local hrp = plr.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return nil, nil end

    -- animal-name lookup: BOTH index names and display names, lowercase keys
    pcall(function()
        if not _MynxxAnimalsData then
            local Datas = ReplicatedStorage:FindFirstChild("Datas")
            _MynxxAnimalsData = Datas and require(Datas:FindFirstChild("Animals"))
        end
        local AD = _MynxxAnimalsData
        if AD and not _MynxxAnimalNames then
            _MynxxAnimalNames = {}
            for idx, info in pairs(AD) do
                local disp = (type(info) == "table" and info.DisplayName) or tostring(idx)
                _MynxxAnimalNames[tostring(idx):lower()] = disp
                _MynxxAnimalNames[tostring(disp):lower()] = disp
            end
        end
    end)

    -- The "Stealing" attribute is the SOURCE OF TRUTH. Without this gate,
    -- anyone standing near a podium got flagged as stealing (stale panel
    -- entries that never refreshed, wrong pet names).
    if plr:GetAttribute("Stealing") ~= true then
        -- steal over: drop the locked session name
        _MynxxStealSession[plr.UserId] = nil
        -- not stealing: refresh the morph snapshot (memo ~1s)
        local mc = _MynxxMorphCache[plr.UserId]
        if _MynxxAnimalNames and (not mc or (os.clock() - mc.t) > 1) then
            local names = {}
            pcall(function()
                for _, m in ipairs(plr.Character:GetDescendants()) do
                    if (m:IsA("Model") or m:IsA("MeshPart")) and _MynxxAnimalNames[m.Name:lower()] then
                        names[m.Name:lower()] = true
                    end
                end
            end)
            _MynxxMorphCache[plr.UserId] = { t = os.clock(), names = names }
        end
        return nil, nil
    end

    local sess = _MynxxStealSession[plr.UserId]
    if type(sess) == "string" then -- old format (pre-update), treat as locked
        sess = { name = sess, locked = true }
        _MynxxStealSession[plr.UserId] = sess
    end

    -- name locked with HIGH confidence (server attribute or carried model)
    -- -> trusted for the whole steal (stable across morphs)
    if sess and sess.locked then
        return sess.owner, sess.name
    end

    -- 1) SERVER ATTRIBUTE first: StealingIndex is set by the server, can't be
    -- faked by a morph and can't hit a neighbour's podium -> most reliable.
    local attrIdx = plr:GetAttribute("StealingIndex")
    if attrIdx ~= nil and tostring(attrIdx) ~= "" then
        local nm = tostring(attrIdx)
        if _MynxxAnimalNames then nm = _MynxxAnimalNames[nm:lower()] or nm end
        local sOwner = sess and sess.owner or nil
        _MynxxStealSession[plr.UserId] = { name = nm, owner = sOwner, locked = true }
        return sOwner, nm
    end

    local morphNames = _MynxxMorphCache[plr.UserId] and _MynxxMorphCache[plr.UserId].names

    -- 2) LIVE carried model: while carrying, the stolen brainrot's model is
    -- parented to the thief's character -- physical truth, locks the session
    -- (and OVERRIDES an earlier podium guess that picked a neighbour's pet).
    -- Models AND MeshParts (the carried visual isn't always a Model), morph
    -- models excluded via the snapshot above.
    local carried
    pcall(function()
        if _MynxxAnimalNames then
            for _, m in ipairs(plr.Character:GetDescendants()) do
                if m:IsA("Model") or m:IsA("MeshPart") then
                    local key = m.Name:lower()
                    local hit = _MynxxAnimalNames[key]
                    if hit and not (morphNames and morphNames[key]) then
                        carried = hit
                        return
                    end
                end
            end
        end
    end)
    if carried then
        local sOwner = sess and sess.owner or nil
        _MynxxStealSession[plr.UserId] = { name = carried, owner = sOwner, locked = true }
        return sOwner, carried
    end

    -- 3) podium scan below is a GUESS -> memoized ~0.4s and kept PROVISIONAL:
    -- re-checked every call so it upgrades as soon as the attribute or the
    -- carried model gives the real name.
    if sess and sess.t and (os.clock() - sess.t) < 0.4 then
        return sess.owner, sess.name
    end

    local isPlayingAnim = false
    local hum = plr.Character:FindFirstChildOfClass("Humanoid")
    local animator = hum and hum:FindFirstChildOfClass("Animator")
    if animator then
        pcall(function()
            for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
                local id = track.Animation and track.Animation.AnimationId
                if id and (id:find("18537363391") or id:find("steal") or id:find("grab")) then
                    isPlayingAnim = true
                    break
                end
            end
        end)
    end

    -- Podium scan: the NEAREST podium wins, occupied or not. If the nearest
    -- podium in range is EMPTY, that's almost certainly the one that was just
    -- grabbed (the server removes its AnimalList entry at grab time) -- in
    -- that case DON'T take a farther occupied neighbour's name (that was the
    -- main source of wrong names on base tags / head ESP / admin panel).
    local plots = Workspace:FindFirstChild("Plots")
    if plots then
        local maxDist = isPlayingAnim and 12 or 4.5
        local Synchronizer
        pcall(function() Synchronizer = _G.__getSync() end)
        -- track BOTH the nearest occupied podium (has an AnimalList entry)
        -- and the nearest empty one; whichever is closer decides below
        local bestOwner, bestName, bestDist = nil, nil, maxDist
        local fbOwner, fbDist = nil, maxDist
        for _, plot in ipairs(plots:GetChildren()) do
            local owner = getPlotOwner(plot)
            if owner == plr then continue end

            local podiums = plot:FindFirstChild("AnimalPodiums")
            if podiums then
                local al
                pcall(function()
                    local ch = Synchronizer and _G.XenSyncGet(plot.Name)
                    al = ch and ch:Get("AnimalList")
                end)
                for _, pod in ipairs(podiums:GetChildren()) do
                    local base = pod:FindFirstChild("Base")
                    local spawn = base and base:FindFirstChild("Spawn")
                    if spawn then
                        local dist = (hrp.Position - spawn.Position).Magnitude
                        local ad = al and (al[pod.Name] or al[tonumber(pod.Name)])
                        if ad and type(ad) == "table" and ad.Index then
                            if dist < bestDist then
                                local animalName = tostring(ad.Index)
                                pcall(function()
                                    if _MynxxAnimalNames then
                                        animalName = _MynxxAnimalNames[animalName:lower()] or animalName
                                    end
                                end)
                                bestDist = dist; bestOwner = owner; bestName = animalName
                            end
                        elseif dist < fbDist then
                            fbDist = dist; fbOwner = owner
                        end
                    end
                end
            end
        end
        if bestOwner and (not fbOwner or bestDist <= fbDist) then
            -- nearest podium is OCCUPIED -> provisional name (upgradable)
            _MynxxStealSession[plr.UserId] = { name = bestName, owner = bestOwner, locked = false, t = os.clock() }
            return bestOwner, bestName
        end
        if fbOwner then
            -- nearest podium is EMPTY -> that's the grabbed one; its name is
            -- unknown from here. Keep any previous provisional name, else stay
            -- generic until the carried model / attribute resolves the truth.
            local nm = (sess and sess.name) or "Brainrot"
            _MynxxStealSession[plr.UserId] = { name = nm, owner = fbOwner, locked = false, t = os.clock() }
            return fbOwner, nm
        end
    end

    -- attribute is true (gated at top) but nothing resolved yet: keep the
    -- previous provisional name for stability, else generic placeholder
    if sess and sess.name then
        return sess.owner, sess.name
    end
    return nil, "Brainrot"
end

-- Keep the morph snapshots fresh even when no UI loop polls a given player
-- (the snapshot is only meaningful if taken BEFORE the steal starts).
-- PERF: paused during TPs and for the first 10s after execute (startup auto
-- TP is the FPS-critical moment).
_G.__MynxxBootClock = _G.__MynxxBootClock or os.clock()
task.spawn(function()
    while true do
        task.wait(1)
        if _G._isTpMoving or (os.clock() - _G.__MynxxBootClock) < 2 then continue end
        for _, p in ipairs(Players:GetPlayers()) do
            if p ~= LocalPlayer and p:GetAttribute("Stealing") ~= true then
                pcall(getStealingInfo, p)
            end
        end
    end
end)

function isOurOverheadBillboard(bb)
    if not bb or not bb.Name then return true end
    local n = bb.Name
    if n:find("^PlayerESP_") or n:find("^BrainrotESP_") or n:find("PlotBeam") then return true end
    if n == "Overhead" or n == "StealProgressBar" or n == "XiAdminPanel" then return true end
    return false
end

_MynxxJTCache = _MynxxJTCache or {} -- [userId] = {t=os.clock(), tag=string|false}
function getPlayerJoinerTag(plr)
    if not plr or not plr.Character then return nil end
    -- 3s memo: this scan (character + PlayerGui + CoreGui descendants) is heavy
    -- and gets called from several 0.5s UI loops per player
    local _c = _MynxxJTCache[plr.UserId]
    if _c and (os.clock() - _c.t) < 3 then return _c.tag or nil end
    local lines, seen = {}, {}
    local function addLine(t)
        if not t or type(t) ~= "string" then return end
        t = t:gsub("^%s+", ""):gsub("%s+$", "")
        if t == "" or #t > 64 then return end
        if t == plr.Name or t == plr.DisplayName then return end
        local key = t:lower()
        if seen[key] then return end
        seen[key] = true
        table.insert(lines, t)
    end
    local function scanBillboard(bb)
        if not bb:IsA("BillboardGui") or not bb.Enabled then return end
        if isOurOverheadBillboard(bb) then return end
        for _, lbl in ipairs(bb:GetDescendants()) do
            if (lbl:IsA("TextLabel") or lbl:IsA("TextBox")) and lbl.Visible ~= false then
                addLine(lbl.Text)
            end
        end
    end
    for _, desc in ipairs(plr.Character:GetDescendants()) do
        if desc:IsA("BillboardGui") then scanBillboard(desc) end
    end
    -- Overhead tags drawn by OUR OTHER local scripts (e.g. the W notifier's
    -- GOOD BOY / BAD BOY labels) live in PlayerGui/CoreGui with Adornee set to
    -- this player's character -- scan those containers too.
    local function scanAdorneed(cont)
        if not cont then return end
        for _, bb in ipairs(cont:GetDescendants()) do
            if bb:IsA("BillboardGui") and bb.Adornee and plr.Character
                and bb.Adornee:IsDescendantOf(plr.Character) then
                scanBillboard(bb)
            end
        end
    end
    pcall(function() scanAdorneed(playerGui) end)
    pcall(function() scanAdorneed(typeof(gethui) == "function" and gethui() or game:GetService("CoreGui")) end)
    pcall(function()
        for attrName, attrVal in pairs(plr:GetAttributes()) do
            if typeof(attrVal) == "string" and #attrVal > 0 and #attrVal <= 40 then
                local an = attrName:lower()
                if an:find("hub") or an:find("script") or an:find("joiner") or an:find("executor") or an:find("tag") then
                    addLine(attrVal)
                end
            end
        end
    end)
    local result = (#lines > 0) and table.concat(lines, " · ") or nil
    _MynxxJTCache[plr.UserId] = { t = os.clock(), tag = result or false }
    return result
end
_G.getPlayerJoinerTag = getPlayerJoinerTag

-- Known-hub classification: normalize raw joiner tags into a "[ X User ]"
-- badge + color. FMLY's overhead tag is "GOOD BOY" (yellow).
HUB_BADGES = {
    -- allowAP = true: this badge does NOT protect -- admin commands stay usable
    { pats = {"bad boy"},                                     badge = "[ BAD BOY ]",   color = Color3.fromRGB(255, 60, 60), allowAP = true },
    { pats = {"good boy", "fmly"}, namePat = "fmly",          badge = "[ FMLY User ]", color = Color3.fromRGB(255, 220, 60) },
    { pats = {"bt user", "bt hub", "bt "},                    badge = "[ BT User ]",   color = Color3.fromRGB(170, 80, 255) },
    { pats = {"w user", "wuser", "w hub", "notifier", "wblox", "w /"}, badge = "[ W User ]", color = Color3.fromRGB(255, 220, 60) },
}
-- Sticky detection cache: overhead tags flicker (billboards get recreated), so
-- once a hub badge is seen on a player it STAYS until they leave the server.
-- This is what makes the AP protection reliable instead of intermittent.
_MynxxHubCache = _MynxxHubCache or {}
-- returns badge, badgeColor, rawTag, allBadges (a player can match SEVERAL hubs;
-- allBadges is the full ordered list so the UI can show them side by side).
-- global on purpose: the main chunk is at Luau's 200-local-register cap
function getHubUserBadge(plr)
    local tag = getPlayerJoinerTag(plr)
    local raw = (tag or ""):lower()
    local nameRaw = ((plr.Name or "") .. " " .. (plr.DisplayName or "")):lower()
    local cached = _MynxxHubCache[plr.UserId]
    if not cached then cached = {}; _MynxxHubCache[plr.UserId] = cached end
    for i, def in ipairs(HUB_BADGES) do
        if not cached[i] then
            for _, pat in ipairs(def.pats) do
                if raw:find(pat, 1, true) then cached[i] = true break end
            end
            if not cached[i] and def.namePat and nameRaw:find(def.namePat, 1, true) then cached[i] = true end
        end
    end
    local list = {}
    local isBadBoy = cached[1] -- HUB_BADGES[1] = BAD BOY
    for i, def in ipairs(HUB_BADGES) do
        if cached[i] then
            -- BAD BOY replaces the FMLY badge entirely (just red [ BAD BOY ])
            if not (isBadBoy and def.badge == "[ FMLY User ]") then
                list[#list + 1] = def
            end
        end
    end
    if #list == 0 then return nil, nil, tag, nil end
    return list[1].badge, list[1].color, tag, list
end
_G.getHubUserBadge = getHubUserBadge
Players.PlayerRemoving:Connect(function(plr) _MynxxHubCache[plr.UserId] = nil end)

function fireClick(button)
    if not button then
        return false
    end

    local ok = pcall(function()
        if typeof(firesignal) == "function" then
            pcall(firesignal, button.MouseButton1Click)
            pcall(firesignal, button.MouseButton1Down)
            pcall(firesignal, button.MouseButton1Up)
            pcall(firesignal, button.Activated)
        end
    end)

    return ok
end
_G.fireClick=fireClick

function runAdminCommand(targetPlayer, commandName)
    -- mark self-targeted commands so the Grief Detector doesn't count them
    pcall(function() if targetPlayer==LocalPlayer then _G.__MynxxSelfCmdT=os.clock() end end)
    if not targetPlayer or not commandName or commandName=="" then return false end
    if isPlayerBlacklisted(targetPlayer) then
        ShowNotification("BLOCKED", targetPlayer.DisplayName .. " is blacklisted")
        return false
    end
    -- Hub users (W / BT / FMLY...) are protected: no admin commands on them,
    -- from ANY path (Admin Panel, Click to AP, Proximity, Spam Base Owner).
    -- EXCEPTION: a [ BAD BOY ] tag lifts the protection -- BUT only for
    -- FMLY/tag-only players. A BT or W user stays protected even as BAD BOY.
    do
        local hubBadge, _, _, hubList = getHubUserBadge(targetPlayer)
        if hubList then
            local badBoy, btOrW = false, false
            for _, def in ipairs(hubList) do
                if def.allowAP then badBoy = true end
                if def.badge == "[ BT User ]" or def.badge == "[ W User ]" then btOrW = true end
            end
            if not badBoy or btOrW then
                ShowNotification("PROTECTED", targetPlayer.DisplayName .. " " .. hubBadge)
                return false
            end
        end
    end
    local realAdminGui=playerGui:FindFirstChild("AdminPanel")
    if not realAdminGui then realAdminGui=playerGui:WaitForChild("AdminPanel",3) end
    if not realAdminGui then return false end
    local wasEnabled = realAdminGui.Enabled
    realAdminGui.Enabled = true
    local okC,contentScroll=pcall(function() return realAdminGui.AdminPanel.Content.ScrollingFrame end)
    if not okC or not contentScroll then realAdminGui.Enabled=wasEnabled; return false end
    local cmdBtn=contentScroll:FindFirstChild(commandName); if not cmdBtn then realAdminGui.Enabled=wasEnabled; return false end
    fireClick(cmdBtn)
    task.wait(0.01)
    local okP,profilesScroll=pcall(function() return realAdminGui.AdminPanel.Profiles.ScrollingFrame end)
    if not okP or not profilesScroll then realAdminGui.Enabled=wasEnabled; return false end
    local playerBtn=profilesScroll:FindFirstChild(targetPlayer.Name)
    if not playerBtn then
        task.wait(0.01)
        playerBtn=profilesScroll:FindFirstChild(targetPlayer.Name)
    end
    if not playerBtn then
        for _,child in ipairs(profilesScroll:GetChildren()) do
            if child:IsA("GuiButton") then local nl=child:FindFirstChildWhichIsA("TextLabel")
                if nl and (nl.Text==targetPlayer.Name or nl.Text==targetPlayer.DisplayName) then playerBtn=child; break end
            end
        end
    end
    if not playerBtn then realAdminGui.Enabled=wasEnabled; return false end
    fireClick(playerBtn)
    apStartCooldown(commandName)
    task.delay(0.05, function()
        if realAdminGui and realAdminGui.Parent then realAdminGui.Enabled=wasEnabled end
    end)
    return true
end
_G.runAdminCommand=runAdminCommand

-- Ragdoll Self autorizado pela própria experiência.
-- O servidor deve validar o jogador antes de aplicar o ragdoll.
function requestRagdollSelf()
    local event = ReplicatedStorage:FindFirstChild("RequestRagdollSelf")
    if not event or not event:IsA("RemoteEvent") then
        pcall(function() ShowNotification("RAGDOLL", "RequestRagdollSelf não encontrado") end)
        return false
    end
    local ok, err = pcall(function() event:FireServer() end)
    if not ok then
        pcall(function() ShowNotification("RAGDOLL", "Falha ao solicitar: " .. tostring(err)) end)
        return false
    end
    return true
end
_G.requestRagdollSelf=requestRagdollSelf


function runAutoBaseActions()
    task.spawn(function()
        task.wait(1.5)
        local char = player.Character or player.CharacterAdded:Wait()
        local hrp = char:WaitForChild("HumanoidRootPart", 10)
        if not hrp then return end
        
        local nearestPlayer = nil
        local nearestDist = math.huge
        local Plots = Workspace:FindFirstChild("Plots")
        if Plots then
            for _, plot in ipairs(Plots:GetChildren()) do
                local owner = getPlotOwner(plot)
                if owner and owner ~= player then
                    local sign = plot:FindFirstChild("PlotSign")
                    if sign then
                        local signPos = (sign:IsA("BasePart") and sign.Position) or (sign.PrimaryPart and sign.PrimaryPart.Position)
                        if signPos then
                            local dist = (hrp.Position - signPos).Magnitude
                            if dist < nearestDist then
                                nearestDist = dist
                                nearestPlayer = owner
                            end
                        end
                    end
                end
            end
        end
        
        if nearestPlayer then
            -- auto base actions removed
        end
    end)
end
_G.runAutoBaseActions = runAutoBaseActions

function isMobyUser(p) return p and p.Character and p.Character:FindFirstChild("_moby_highlight")~=nil end
function isKawaifuUser(p) return p and p.Character and p.Character:FindFirstChild("KaWaifu_NeonHighlight")~=nil end

function getNextAvailableCommand()
    local priorityCmds={"ragdoll","balloon","rocket","jail"}
    for _,cmd in ipairs(priorityCmds) do if not apIsOnCooldown(cmd) then return cmd end end
    for _,cmd in ipairs(AP_ALL_COMMANDS) do if not apIsOnCooldown(cmd) then return cmd end end
    return nil
end

function kickPlayer(stolenText)
    local isAutoKickSteal = false
    
    if stolenText and type(stolenText) == "string" then
        isAutoKickSteal = true
    end

    if Config.KickToPrivateServer and PrivateServerCode and PrivateServerCode ~= "" and isAutoKickSteal then
        task.delay(0.2, function()
            pcall(function()
                local ExperienceService = game:GetService("ExperienceService")
                ExperienceService:LaunchExperience({
                    placeId = game.PlaceId,
                    linkCode = PrivateServerCode,
                })
            end)
        end)
        return
    end
    pcall(function() game:Shutdown() end)
    pcall(function() LocalPlayer:Kick("\nMynxx") end)
end

-- SHARED STATE
SharedState = {SelectedPetData=nil, AllAnimalsCache={}, ListNeedsRedraw=true, InitialScanComplete=false, seenUIDs={}, BrainrotNames={}}

-- NOTIFICATION SYSTEM
function ShowNotification(title, text)
    -- Disabled: all notifications removed by user request
    return
end

-- ============================================================
-- RESET
-- ============================================================

do
--[[
    Roblox Instant Reset & Lagback Detector (Defensive Always-Working Hook)
    Created by Antigravity (DeepMind Coding Assistant)
    
    Fixes:
    - Reverted capture-state resetting. Once TargetRemote is captured, it is kept for the entire session.
    - Fixed the reset-spam issue. Resetting state resets immediately upon character removal so you can spam resets.
--]]

-- SERVICES
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
ReplicatedStorage = game:GetService("ReplicatedStorage")
Workspace = game:GetService("Workspace")
StarterGui = game:GetService("StarterGui")
UserInputService = game:GetService("UserInputService")

local LocalPlayer = Players.LocalPlayer

-- CONFIGURATION DEFAULT
local Config = {
    Enabled = true,
    DetectionThreshold = 6,
    HistoryDuration = 2.0,
    VelocityDotThreshold = -0.3,
    
    ResetKey = "NONE",           -- Managed via main Keybinds (default X)
    FloodCount = 50,             -- Number of junk fires to force server lagback reset
    
    ResetMethods = {
        SpamHeartbeat = true,    -- Spam captured ToolActivationController remote
        HumanoidHealth = true,   -- Fallback: Set health to 0
        BreakJoints = true,      -- Fallback: Break joints
        ChangeStateDead = true,  -- Fallback: Force dead state
        DestroyHumanoid = false, -- Fallback: Destroy humanoid
        FireRemotes = true       -- Fallback: Fire detected generic reset remotes
    },
    
    ScanInterval = 15,
    RemoteKeywords = {"reset", "kill", "die", "suicide", "respawn", "clear", "death", "damage", "reavatar"},
    BlacklistedRemotes = {"AdminReset", "StaffReset", "ModReset"},
    
    Debug = false,
    NotifyOnReset = true
}

-- STATE VARIABLES
TargetRemote = nil -- Dynamically captured heartbeat remote
rawFS = nil        -- Original unhooked FireServer function
resetRemotes = {}
positionHistory = {}
lastPosition = nil
lastVelocity = Vector3.new(0, 0, 0)
local isTeleporting = false
resetting = false
scanTimer = 0
lastHistoryUpdate = 0

-- LOGGING UTILITY
function log(message, isWarn)
    -- Disabled: do not print or warn anything to dev console
    return
end

-- EXPORTED TELEPORT BYPASS API
_G.NotifyLocalTeleport = function(duration)
    local delayTime = duration or 0.5
    isTeleporting = true
    task.delay(delayTime, function()
        isTeleporting = false
    end)
end
shared.NotifyLocalTeleport = _G.NotifyLocalTeleport

-- OPTIMIZED INCREMENTAL REMOTE EVENT SCANNER (For Generic Fallbacks)
function scanForRemotes()
    table.clear(resetRemotes)
    
    task.spawn(function()
        local searchAreas = {ReplicatedStorage, Workspace}
        local playerGui = LocalPlayer:FindFirstChildOfClass("PlayerGui")
        if playerGui then
            table.insert(searchAreas, playerGui)
        end
        
        local count = 0
        for _, area in ipairs(searchAreas) do
            local queue = {area}
            while #queue > 0 do
                local current = table.remove(queue, 1)
                count = count + 1
                if count % 150 == 0 then
                    task.wait()
                end
                
                pcall(function()
                    for _, child in ipairs(current:GetChildren()) do
                        if child:IsA("RemoteEvent") or child:IsA("RemoteFunction") then
                            local nameLower = child.Name:lower()
                            local isMatch = false
                            for _, keyword in ipairs(Config.RemoteKeywords) do
                                if string.find(nameLower, keyword, 1, true) then
                                    isMatch = true
                                    break
                                end
                            end
                            
                            if isMatch then
                                local isBlacklisted = false
                                for _, black in ipairs(Config.BlacklistedRemotes) do
                                    if string.find(child.Name, black, 1, true) then
                                        isBlacklisted = true
                                        break
                                    end
                                end
                                
                                if not isBlacklisted then
                                    table.insert(resetRemotes, child)
                                end
                            end
                        elseif child:IsA("Folder") or child:IsA("Configuration") or child:IsA("Model") or child:IsA("Tool") then
                            table.insert(queue, child)
                        end
                    end
                end)
            end
        end
    end)
end

-- --- DYNAMIC REMOTE INTERCEPTION (ToolActivationController Heartbeat Hook) ---
function hookHeartbeatRemote()
    local newcc = newcclosure or function(f) return f end
    
    pcall(function()
        local seen = {}
        local capturing = true
        
        local function handleFire(self, ...)
            if not capturing then return end
            if not self:IsA("RemoteEvent") then return end
            
            local a1 = (select("#", ...) >= 1) and (select(1, ...)) or nil
            if typeof(a1) ~= "string" then return end
            
            local nm = self.Name
            if type(nm) ~= "string" or not nm:match("^RE/%x%x%x%x%x%x%x%x") then return end
            
            local callingScript = nil
            pcall(function() callingScript = getcallingscript() end)
            
            local isScriptMatch = false
            if callingScript then
                if callingScript.Name == "ToolActivationController" or string.find(callingScript.Name, "ToolActivationController") then
                    isScriptMatch = true
                end
            else
                isScriptMatch = true
            end
            
            if isScriptMatch then
                local c = (seen[self] or 0) + 1
                seen[self] = c
                if c >= 2 then
                    TargetRemote = self
                    capturing = false
                    seen = nil
                    log("ToolActivationController heartbeat remote captured: " .. self:GetFullName())
                end
            end
        end

        -- Hooks de FireServer removidos: o scanner não intercepta nem altera
        -- chamadas de rede do jogo.
        log("Remote scanner em modo passivo; nenhum FireServer foi hookado.")
    end)
end

-- FTUE TUTORIAL ARROW REPAIR
-- Fresh accounts: the game's FTUEController indexes TutorialArrow.Start every
-- frame (Line 221). When "Start" is missing (stripped before the IsProtected
-- fix, or the hub blew through the tutorial steps so it never got built) the
-- console error-spams forever -- especially visible right after a reset, when
-- the FTUE re-points the arrow. IsProtected only PREVENTS future strips; this
-- block REPAIRS the arrow live: recreate invisible Start/End parts whenever
-- they're missing, and re-add them if anything deletes them again.
task.spawn(function()
    local function ensureBeamKit(part)
        -- Line 221 indexes DEEPER than the bare part: first error was
        -- TutorialArrow.Start, the next one was Start.Beam. So each repaired
        -- part also carries an Attachment + a disabled Beam wired to it, so
        -- further member accesses resolve instead of error-spamming.
        local a = part:FindFirstChildOfClass("Attachment")
        if not a then
            a = Instance.new("Attachment")
            a.Name = "Attachment"
            a.Parent = part
        end
        if not part:FindFirstChild("Beam") then
            local b = Instance.new("Beam")
            b.Name = "Beam"
            b.Enabled = false
            b.Attachment0 = a
            b.Attachment1 = a
            b.Parent = part
        end
    end
    local function repairArrow(model)
        if not model or not model.Parent or not model:IsA("Model") then return end
        for _, nm in ipairs({ "Start", "End" }) do
            local p = model:FindFirstChild(nm)
            if not p then
                p = Instance.new("Part")
                p.Name = nm
                p.Size = Vector3.new(0.1, 0.1, 0.1)
                p.Transparency = 1
                p.Anchored = true
                p.CanCollide = false
                pcall(function() p.CanQuery = false; p.CanTouch = false end)
                pcall(function() p.CFrame = model:GetPivot() end)
                p.Parent = model
            end
            if p:IsA("BasePart") then pcall(ensureBeamKit, p) end
        end
    end
    local function hookArrow(inst)
        if inst.Name ~= "TutorialArrow" or not inst:IsA("Model") then return end
        repairArrow(inst)
        inst.ChildRemoved:Connect(function() task.defer(repairArrow, inst) end)
        inst.DescendantRemoving:Connect(function() task.defer(repairArrow, inst) end)
    end
    local cur = Workspace:FindFirstChild("TutorialArrow")
    if cur then hookArrow(cur) end
    Workspace.ChildAdded:Connect(hookArrow)

    -- LAST-RESORT SILENCER: if the FTUEController STILL error-spams about
    -- TutorialArrow (some member we didn't predict), disable its per-frame
    -- RunService connections outright -- same getconnections+getinfo
    -- technique as Strip(). Client-side tutorial visuals only, and it only
    -- triggers off the actual error, so healthy sessions never touch it.
    local _ftueSilenced = false
    local function silenceFTUE()
        if _ftueSilenced then return end
        if not (getconnections and getinfo) then return end
        _ftueSilenced = true
        local RS = game:GetService("RunService")
        for _, sig in ipairs({ RS.RenderStepped, RS.Heartbeat, RS.Stepped }) do
            pcall(function()
                for _, c in ipairs(getconnections(sig)) do
                    local f = c.Function
                    if f and c.Enabled then
                        local ok, info = pcall(getinfo, f)
                        if ok and info and tostring(info.source):find("FTUEController") then
                            pcall(function() c:Disable() end)
                        end
                    end
                end
            end)
        end
    end
    pcall(function()
        game:GetService("LogService").MessageOut:Connect(function(msg, msgType)
            if msgType == Enum.MessageType.MessageError and tostring(msg):find("TutorialArrow", 1, true) then
                task.defer(silenceFTUE)
            end
        end)
    end)
end)

-- Reset local e único. Não usa remotes, hooks, payloads ou loops de spam.
-- Em uma experiência própria, o servidor deve tratar a morte/respawn através
-- do fluxo oficial do jogo; no cliente, esta rotina apenas solicita o estado
-- Dead e ajusta a vida local quando permitido.
function instantReset()
    if instaResetCooldown then return false end
    instaResetCooldown = true
    task.spawn(function()
        local char = LocalPlayer.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if hum and hum.Parent then
            pcall(function() hum:ChangeState(Enum.HumanoidStateType.Dead) end)
            pcall(function() hum.Health = 0 end)
            pcall(function() char:BreakJoints() end)
        end
        task.wait(0.35)
        instaResetCooldown = false
    end)
    return true
end

-- Regenerar como o botão oficial do Roblox: desativa temporariamente o
-- AntiDie, mata o Humanoid e quebra os joints para o servidor criar um novo
-- Character no SpawnLocation da própria base. Não faz teleport manual.
function regenerateAtBase()
    if instaResetCooldown then return false end
    instaResetCooldown = true
    _G.AntiDieDisabled = true

    local oldCharacter = LocalPlayer.Character
    local oldHumanoid = oldCharacter and oldCharacter:FindFirstChildOfClass("Humanoid")
    if not oldHumanoid then
        instaResetCooldown = false
        _G.AntiDieDisabled = false
        return false
    end

    local respawned = false
    local characterConnection
    characterConnection = LocalPlayer.CharacterAdded:Connect(function(newCharacter)
        if newCharacter ~= oldCharacter then
            respawned = true
            if characterConnection then characterConnection:Disconnect() end
        end
    end)

    pcall(function() oldHumanoid.BreakJointsOnDeath = true end)
    pcall(function() oldHumanoid:ChangeState(Enum.HumanoidStateType.Dead) end)
    pcall(function() oldHumanoid.Health = 0 end)
    task.defer(function() pcall(function() oldCharacter:BreakJoints() end) end)

    task.spawn(function()
        local deadline = os.clock() + 6
        while not respawned and os.clock() < deadline do task.wait(0.1) end
        if characterConnection then characterConnection:Disconnect() end
        _G.AntiDieDisabled = false
        instaResetCooldown = false
    end)
    return true
end
_G.MynxxRegenerate = regenerateAtBase

-- Não substitui o ResetButtonCallback por um BindableEvent customizado.
-- O botão oficial fica sob controle do Roblox/servidor e não dispara remotes
-- desconhecidos nem interfere no fluxo normal de respawn.
task.spawn(function()
    for _ = 1, 12 do
        local ok = pcall(function()
            game:GetService("StarterGui"):SetCore("ResetButtonCallback", true)
        end)
        if ok then break end
        task.wait(1)
    end
end)

-- KEYBIND TRIGGER (Disabled local listener to avoid collision with 'Drop Brainrot' keybind G)
-- Reset is managed by the main Keybinds setting in the GUI (defaults to key X)

_G.InstantReset = regenerateAtBase -- Global access for other scripts

-- HISTORY CACHE MANAGEMENT
function updateHistory(pos)
    local now = tick()
    if now - lastHistoryUpdate < 0.05 then return end
    lastHistoryUpdate = now
    
    table.insert(positionHistory, {pos = pos, time = now})
    
    while #positionHistory > 0 and now - positionHistory[1].time > Config.HistoryDuration do
        table.remove(positionHistory, 1)
    end
end

-- --- MAIN MONITORING LOOP ---
function onHeartbeat(dt)
    if resetting then return end
    
    -- Scan remotes periodically in background
    scanTimer = scanTimer + dt
    if scanTimer >= Config.ScanInterval then
        scanTimer = 0
        scanForRemotes()
    end
end

-- INITIALIZATION
scanForRemotes()
-- hookHeartbeatRemote() removed -- it stacked a second FireServer hook on top of
-- the Vanish bypass hook, which broke the chain after re-injects and stopped
-- in-game tool activation from working. The Vanish FireServer hook at the top of
-- the file is the ONLY hook this script installs now.
RunService.Heartbeat:Connect(onHeartbeat)



-- Compatibility wrapper for executeReset
lastBalloonResetTime = 0
executeReset = function(isBalloon)
    if isBalloon then
        if tick() - lastBalloonResetTime < 20 then return end
        lastBalloonResetTime = tick()
    end
    instantReset()
end
_G.executeReset = executeReset
end

-- CLONE
function instantClone()
    local c=player.Character; if not c then return end
    local h=c:FindFirstChildOfClass("Humanoid"); if not h then return end
    local cl=player.Backpack:FindFirstChild("Quantum Cloner") or c:FindFirstChild("Quantum Cloner"); if not cl then return end
    pcall(function() h:UnequipTools() end); task.wait()
    if cl.Parent~=c then h:EquipTool(cl); task.wait() end
    local tf=playerGui:FindFirstChild("ToolsFrames"); local qc=tf and tf:FindFirstChild("QuantumCloner")
    local tb=qc and qc:FindFirstChild("TeleportToClone"); if not tb then return end
    _G.isCloning=true; cl:Activate(); task.wait(0.05); tb.Visible=true
    pcall(function() firesignal(tb.MouseButton1Click) end)
    pcall(function() firesignal(tb.MouseButton1Up) end)
    pcall(function() firesignal(tb.Activated) end)
    task.delay(0.55, function() _G.isCloning=false end)
end

-- DROP BRAINROT
local _wfConns,_wfActive={},false
local function stopWalkFling() _wfActive=false; for _,c in ipairs(_wfConns) do if typeof(c)=="RBXScriptConnection" then c:Disconnect() end end; _wfConns={} end
local function startWalkFling()
    _wfActive=true; local ch=player.Character; if not ch then return end
    local rr=ch:FindFirstChild("HumanoidRootPart")
    for _,o in pairs(Workspace.CurrentCamera:GetChildren()) do if o.Name=="HumanoidRootPart" then rr=o; break end end
    if not rr then return end
    table.insert(_wfConns,RunService.Stepped:Connect(function() if not _wfActive then return end
        for _,p in ipairs(Players:GetPlayers()) do if p~=player and p.Character then for _,pt in ipairs(p.Character:GetChildren()) do if pt:IsA("BasePart") then pt.CanCollide=false end end end end
    end))
    local co=coroutine.create(function()
        if _G.invisibleStealEnabled then rr.CFrame=rr.CFrame*CFrame.new(0,3,0) end
        while _wfActive do RunService.Heartbeat:Wait(); if not rr or not rr.Parent then break end
            local v=rr.Velocity; rr.Velocity=v*10000+Vector3.new(0,10000,0)
            RunService.RenderStepped:Wait(); if rr then rr.Velocity=v end
            RunService.Stepped:Wait(); if rr then rr.Velocity=v+Vector3.new(0,0.1,0) end
        end
    end); coroutine.resume(co); table.insert(_wfConns,co)
end
local function runDropBrainrot() if _wfActive then return end; startWalkFling(); task.delay(0.4,stopWalkFling) end
-- ============================================================
-- INVISIBLE STEAL
-- ============================================================
do
animPlaying = false
tracks = {}
folderConnections = {}
serverGhosts = {}
ghostEnabled = true
lagbackCallCount = 0
lagbackWindowStart = 0
lastLagbackTime = 0
errorOrbActive = false
errorOrb = nil
errorOrbConnection = nil

_G.invisibleStealEnabled = false
_G.InvisStealAngle = Config.InvisStealAngle or 225
_G.SinkSliderValue = Config.SinkSliderValue or 7
_G.AutoRecoverLagback = Config.AutoRecoverLagback ~= nil and Config.AutoRecoverLagback or true
_G.AutoInvisDuringSteal = Config.AutoInvisDuringSteal or false

function clearErrorOrb()
    if errorOrb and errorOrb.Parent then errorOrb:Destroy() end
    errorOrb = nil; errorOrbActive = false
    if errorOrbConnection then errorOrbConnection:Disconnect(); errorOrbConnection = nil end
end

function createErrorOrb()
    if errorOrbActive then return end
    errorOrbActive = true
    for _, ghost in pairs(serverGhosts) do if ghost and ghost.Parent then ghost:Destroy() end end
    serverGhosts = {}
    -- Error UI notifications disabled
end

function createServerGhost(position)
    if not ghostEnabled or errorOrbActive then return end
    local now = tick()
    if now - lastLagbackTime < 0.05 then return end
    lastLagbackTime = now
    if now - lagbackWindowStart > 1 then lagbackCallCount = 0; lagbackWindowStart = now end
    lagbackCallCount = lagbackCallCount + 1
    if lagbackCallCount >= 7 then createErrorOrb(); return end
    for _, g in pairs(serverGhosts) do if g and g.Parent then g:Destroy() end end
    serverGhosts = {}
    local ghost = Instance.new("Part")
    ghost.Name = "LagbackGhost"; ghost.Shape = Enum.PartType.Ball
    ghost.Size = Vector3.new(3, 3, 3); ghost.Color = Color3.fromRGB(255, 0, 0)
    ghost.Material = Enum.Material.Glass; ghost.Transparency = 0.3
    ghost.CanCollide = false; ghost.Anchored = true; ghost.CastShadow = false
    ghost.Position = position + Vector3.new(0, 5, 0); ghost.Parent = Workspace.CurrentCamera
    table.insert(serverGhosts, ghost)
    -- Lagback text notifications & billboard GUIs disabled
end

function clearAllGhosts()
    for _, ghost in pairs(serverGhosts) do pcall(function() if ghost and ghost.Parent then ghost:Destroy() end end) end
    serverGhosts = {}; clearErrorOrb(); lagbackCallCount = 0; lastLagbackTime = 0
    pcall(function()
        local pg = player:FindFirstChild("PlayerGui")
        if pg then for _, gui in pairs(pg:GetChildren()) do if gui.Name == "LagbackNotification" then gui:Destroy() end end end
    end)
    pcall(function() if Workspace.CurrentCamera then for _, c in pairs(Workspace.CurrentCamera:GetChildren()) do if c.Name == "LagbackGhost" then c:Destroy() end end end end)
    pcall(function() for _, c in pairs(Workspace:GetDescendants()) do if c.Name == "LagbackGhost" then c:Destroy() end end end)
end

function removeFolders()
    local pf = Workspace:FindFirstChild(player.Name)
    if not pf then return end
    local dr = pf:FindFirstChild("DoubleRig")
    if dr then
        local rr = dr:FindFirstChild("HumanoidRootPart") or dr:FindFirstChildWhichIsA("BasePart")
        if rr and ghostEnabled then createServerGhost(rr.Position) end
        dr:Destroy()
    end
    local cs = pf:FindFirstChild("Constraints")
    if cs then cs:Destroy() end
    local conn = pf.ChildAdded:Connect(function(child)
        if child.Name == "DoubleRig" then
            task.defer(function()
                local rr = child:FindFirstChild("HumanoidRootPart") or child:FindFirstChildWhichIsA("BasePart")
                if rr and ghostEnabled then createServerGhost(rr.Position) end
                child:Destroy()
            end)
        elseif child.Name == "Constraints" then child:Destroy() end
    end)
    table.insert(folderConnections, conn)
end

function doClone()
    local character = player.Character
    if character and character:FindFirstChild("Humanoid") and character.Humanoid.Health > 0 then
        hip = character.Humanoid.HipHeight
        oldRoot = character:FindFirstChild("HumanoidRootPart")
        if not oldRoot or not oldRoot.Parent then return false end
        for _, c in pairs(oldRoot:GetChildren()) do
            if c:IsA("Attachment") and (c.Name:find("Beam") or c.Name:find("Attach")) then c:Destroy() end
        end
        for _, c in pairs(oldRoot:GetChildren()) do if c:IsA("Beam") then c:Destroy() end end
        local tmp = Instance.new("Model"); tmp.Parent = game
        character.Parent = tmp
        clone = oldRoot:Clone(); clone.Parent = character
        oldRoot.Parent = Workspace.CurrentCamera
        clone.CFrame = oldRoot.CFrame; character.PrimaryPart = clone
        character.Parent = Workspace
        for _, v in pairs(character:GetDescendants()) do
            if v:IsA("Weld") or v:IsA("Motor6D") then
                if v.Part0 == oldRoot then v.Part0 = clone end
                if v.Part1 == oldRoot then v.Part1 = clone end
            end
        end
        tmp:Destroy(); return true
    end
    return false
end

function revertClone()
    local character = player.Character
    if not oldRoot or not oldRoot:IsDescendantOf(Workspace) or not character or character.Humanoid.Health <= 0 then return end
    local tmp = Instance.new("Model"); tmp.Parent = game
    character.Parent = tmp
    oldRoot.Parent = character; character.PrimaryPart = oldRoot
    character.Parent = Workspace; oldRoot.CanCollide = true
    for _, v in pairs(character:GetDescendants()) do
        if v:IsA("Weld") or v:IsA("Motor6D") then
            if v.Part0 == clone then v.Part0 = oldRoot end
            if v.Part1 == clone then v.Part1 = oldRoot end
        end
    end
    if clone then local p = clone.CFrame; clone:Destroy(); clone = nil; oldRoot.CFrame = p end
    oldRoot = nil
    if character and character.Humanoid then character.Humanoid.HipHeight = hip end
    clearAllGhosts()
end

function animationTrickery()
    local character = player.Character
    if character and character:FindFirstChild("Humanoid") and character.Humanoid.Health > 0 then
        local anim = Instance.new("Animation")
        anim.AnimationId = "http://www.roblox.com/asset/?id=18537363391"
        local humanoid = character.Humanoid
        local animator = humanoid:FindFirstChild("Animator") or Instance.new("Animator", humanoid)
        local animTrack = animator:LoadAnimation(anim)
        animTrack.Priority = Enum.AnimationPriority.Action4
        animTrack:Play(0, 1, 0); anim:Destroy()
        table.insert(tracks, animTrack)
        animTrack.Stopped:Connect(function() if animPlaying then animationTrickery() end end)
        task.delay(0, function()
            animTrack.TimePosition = 0.7
            task.delay(0.3, function() if animTrack then animTrack:AdjustSpeed(math.huge) end end)
        end)
    end
end

_invisToggleCooldown = 0
_G.InvisStealAngle = Config.InvisStealAngle or 180
_G.SinkSliderValue = Config.SinkSliderValue or 8

local serverGhosts = {}
local lagbackCallCount = 0
local lagbackWindowStart = 0
local lastLagbackTime = 0
local errorOrbActive = false

local function createErrorOrb()
    if errorOrbActive then return end
    errorOrbActive = true
    local orb = Instance.new("Part")
    orb.Name = "InvisErrorOrb"
    orb.Shape = Enum.PartType.Ball
    orb.Size = Vector3.new(4, 4, 4)
    orb.Color = Color3.fromRGB(255, 0, 0)
    orb.Material = Enum.Material.Neon
    orb.Transparency = 0.2
    orb.CanCollide = false
    orb.Anchored = true
    orb.CastShadow = false
    orb.Position = (player.Character and player.Character:FindFirstChild("HumanoidRootPart") and player.Character.HumanoidRootPart.Position + Vector3.new(0, 8, 0)) or Vector3.new(0, 100, 0)
    orb.Parent = Workspace.CurrentCamera
    local bb = Instance.new("BillboardGui", orb)
    bb.Size = UDim2.new(0, 200, 0, 50)
    bb.StudsOffset = Vector3.new(0, 4, 0)
    bb.AlwaysOnTop = true
    local lbl = Instance.new("TextLabel", bb)
    lbl.Size = UDim2.new(1, 0, 1, 0)
    lbl.BackgroundTransparency = 1
    lbl.Text = "LAGBACK DETECTED\nDISABLE INVIS STEAL"
    lbl.TextColor3 = Color3.fromRGB(255, 0, 0)
    lbl.TextStrokeTransparency = 0
    lbl.TextStrokeColor3 = Color3.new(0, 0, 0)
    lbl.Font = Enum.Font.GothamBold
    lbl.TextScaled = true
    task.delay(2, function()
        if orb then orb:Destroy() end
        errorOrbActive = false
    end)
end

local function clearErrorOrb()
    errorOrbActive = false
    local cam = Workspace.CurrentCamera
    if cam then
        for _, c in pairs(cam:GetChildren()) do
            if c.Name == "InvisErrorOrb" then c:Destroy() end
        end
    end
end

local function createServerGhost(position)
    if not ghostEnabled or errorOrbActive then return end
    local now = tick()
    if now - lastLagbackTime < 0.05 then return end
    lastLagbackTime = now
    if now - lagbackWindowStart > 1 then lagbackCallCount = 0; lagbackWindowStart = now end
    lagbackCallCount += 1
    if lagbackCallCount >= 7 then createErrorOrb(); return end
    for _, g in pairs(serverGhosts) do if g and g.Parent then g:Destroy() end end
    serverGhosts = {}
    local ghost = Instance.new("Part")
    ghost.Name = "LagbackGhost"
    ghost.Shape = Enum.PartType.Ball
    ghost.Size = Vector3.new(3, 3, 3)
    ghost.Color = Color3.fromRGB(255, 0, 0)
    ghost.Material = Enum.Material.Glass
    ghost.Transparency = 0.3
    ghost.CanCollide = false
    ghost.Anchored = true
    ghost.CastShadow = false
    ghost.Position = position + Vector3.new(0, 5, 0)
    ghost.Parent = Workspace.CurrentCamera
    local bb = Instance.new("BillboardGui", ghost)
    bb.Size = UDim2.new(0, 400, 0, 60)
    bb.StudsOffset = Vector3.new(0, 4, 0)
    bb.AlwaysOnTop = true
    local bl = Instance.new("TextLabel", bb)
    bl.Size = UDim2.new(1, 0, 0, 25)
    bl.BackgroundTransparency = 1
    bl.Text = "LAGBACK DETECTED"
    bl.TextColor3 = Color3.fromRGB(255, 0, 0)
    bl.TextStrokeTransparency = 0
    bl.TextStrokeColor3 = Color3.new(0, 0, 0)
    bl.Font = Enum.Font.GothamBold
    bl.TextScaled = true
    local bw = Instance.new("TextLabel", bb)
    bw.Size = UDim2.new(1, 0, 0, 25)
    bw.Position = UDim2.new(0, 0, 0, 25)
    bw.BackgroundTransparency = 1
    bw.Text = "DISABLE INVISIBLE STEAL NOW"
    bw.TextColor3 = Color3.fromRGB(200, 200, 200)
    bw.TextStrokeTransparency = 0
    bw.TextStrokeColor3 = Color3.new(0, 0, 0)
    bw.Font = Enum.Font.GothamBold
    bw.TextScaled = true
    table.insert(serverGhosts, ghost)
end

local function clearAllGhosts()
    for _, ghost in pairs(serverGhosts) do pcall(function() if ghost and ghost.Parent then ghost:Destroy() end end) end
    serverGhosts = {}
    lagbackCallCount = 0
    lastLagbackTime = 0
    lagbackWindowStart = 0
    clearErrorOrb()
    pcall(function()
        local pg = player:FindFirstChild("PlayerGui")
        if pg then for _, gui in pairs(pg:GetChildren()) do if gui.Name == "LagbackNotification" then gui:Destroy() end end end
    end)
    pcall(function()
        local cam = Workspace.CurrentCamera
        if cam then for _, c in pairs(cam:GetChildren()) do if c.Name == "LagbackGhost" then c:Destroy() end end end
    end)
    pcall(function()
        for _, c in pairs(Workspace:GetDescendants()) do
            if c.Name == "LagbackGhost" then c:Destroy() end
        end
    end)
end

local function calculateAutoRotation()
    if not _G.AutoRotateInvis then return _G.InvisStealAngle or 180 end
    local char = player.Character
    if not char then return 180 end
    local minY, maxY = math.huge, -math.huge
    local maxHorizontal = 0
    for _, part in pairs(char:GetDescendants()) do
        if part:IsA("BasePart") and part ~= clone then
            local size = part.Size
            local pos = part.Position
            minY = math.min(minY, pos.Y - size.Y/2)
            maxY = math.max(maxY, pos.Y + size.Y/2)
            maxHorizontal = math.max(maxHorizontal, size.X, size.Z)
        end
    end
    local modelHeight = maxY - minY
    if modelHeight <= 0 then return 180 end
    local sinkAmount = (_G.SinkSliderValue or 5) * 0.5
    local angle
    if sinkAmount >= modelHeight then
        angle = 180
    else
        local ratio = math.min(1, sinkAmount / modelHeight)
        angle = 180 + math.deg(math.asin(1 - ratio)) + 15
        angle = math.clamp(angle, 180, 270)
    end
    return angle
end

function invisTurnOff()
    clearAllGhosts()
    if not animPlaying then return end
    local character = player.Character
    local humanoid = character and character:FindFirstChildOfClass("Humanoid")
    animPlaying = false; _G.invisibleStealEnabled = false
    setToggle("Invisible Steal", false)
    for _, t in pairs(tracks) do pcall(function() t:Stop(0) end) end
    tracks = {}
    if connection then connection:Disconnect(); connection = nil end
    for _, c in ipairs(folderConnections) do if c then c:Disconnect() end end
    folderConnections = {}
    revertClone(); clearAllGhosts()
    -- Force reset all animations back to default
    if humanoid then
        pcall(function()
            local animator = humanoid:FindFirstChildOfClass("Animator")
            if animator then
                for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
                    if track.Priority == Enum.AnimationPriority.Action4 or track.Priority == Enum.AnimationPriority.Action3 then
                        track:Stop(0)
                    end
                end
            end
            humanoid:ChangeState(Enum.HumanoidStateType.GettingUp)
            task.defer(function()
                if humanoid and humanoid.Parent then
                    humanoid:ChangeState(Enum.HumanoidStateType.Running)
                end
            end)
        end)
    end
    if _G.updateMovementPanelInvisVisual then pcall(_G.updateMovementPanelInvisVisual, false) end
    -- Instantly disable walkspeed when invis steal turns off
    if WalkSpeedState and WalkSpeedState.enabled and Config.WalkSpeedEnabled then
        setWalkSpeedEnabled(false)
    end
    _invisToggleCooldown = tick()
end

function invisTurnOn()
    if animPlaying then return end
    local character = player.Character
    if not character then return end
    local humanoid = character:FindFirstChildOfClass("Humanoid")
    if not humanoid then return end
    animPlaying = true; _G.invisibleStealEnabled = true
    setToggle("Invisible Steal", true)
    if _G.updateMovementPanelInvisVisual then pcall(_G.updateMovementPanelInvisVisual, true) end
    tracks = {}; removeFolders()
    local success = doClone()
    if success then
        task.wait(0.05); animationTrickery()
        task.defer(function()
            if _G.resetBrainrotBeam then pcall(_G.resetBrainrotBeam) end
            if _G.resetPlotBeam then pcall(_G.resetPlotBeam) end
            task.wait(0.1)
            if _G.updateBrainrotBeam then pcall(_G.updateBrainrotBeam) end
            if _G.createPlotBeam then pcall(_G.createPlotBeam) end
        end)
        -- Enable walkspeed 1 second after invis steal activates
        task.delay(1, function()
            if _G.invisibleStealEnabled and not WalkSpeedState.enabled then
                setWalkSpeedEnabled(true)
            end
        end)
        local lastSetPosition = nil; local skipFrames = 5
        connection = RunService.PreSimulation:Connect(function()
            if character and character:FindFirstChild("Humanoid") and character.Humanoid.Health > 0 and oldRoot then
                local root = character.PrimaryPart or character:FindFirstChild("HumanoidRootPart")
                if root then
                    if skipFrames > 0 then skipFrames = skipFrames - 1; lastSetPosition = nil
                    elseif lastSetPosition and ghostEnabled then
                        local currentPos = oldRoot.Position
                        local jumpDist = (currentPos - lastSetPosition).Magnitude
                        if jumpDist > 6 and not _G.RecoveryInProgress and player:GetAttribute("Stealing") then
                            lastSetPosition = nil; createServerGhost(currentPos)
                            if _G.AutoRecoverLagback and _G._forceInvisToggle then
                                _G.RecoveryInProgress = true
                                task.spawn(function()
                                    pcall(_G._forceInvisToggle); task.wait(0.6)
                                    if player:GetAttribute("Stealing") then
                                        pcall(_G._forceInvisToggle)
                                    end
                                    _G.RecoveryInProgress = false
                                end)
                            end
                        end
                    end
                    if clone then clone.CanCollide = true end
                    if oldRoot and oldRoot.Parent then
                        for _, c in pairs(oldRoot:GetChildren()) do
                            if c:IsA("Attachment") or c:IsA("Beam") then c:Destroy() end
                        end
                        local sa = (_G.SinkSliderValue or 7) * 0.5
                        local cf = root.CFrame - Vector3.new(0, sa, 0)
                        local rotAngle = _G.AutoRotateInvis and calculateAutoRotation() or (_G.InvisStealAngle or 225)
                        oldRoot.CFrame = cf * CFrame.Angles(math.rad(rotAngle), 0, 0)
                        oldRoot.AssemblyLinearVelocity = root.AssemblyLinearVelocity; oldRoot.CanCollide = false
                        lastSetPosition = oldRoot.Position
                    end
                end
            end
        end)
    end
end

_G.toggleInvisibleSteal = function()
    if (tick() - _invisToggleCooldown) < 0.3 then return end
    if animPlaying then invisTurnOff() else invisTurnOn() end
end

-- Force toggle that bypasses debounce (for auto-recover and auto-invis)
_G._forceInvisToggle = function()
    if animPlaying then invisTurnOff() else invisTurnOn() end
end

player.CharacterAdded:Connect(function(newChar)
    task.wait(0.1)
    if Config then Config.ClickToAP = false end
    clearErrorOrb(); clearAllGhosts(); lagbackCallCount = 0
    pcall(function() for _, c in pairs(Workspace.CurrentCamera:GetChildren()) do if c:IsA("BasePart") and c.Name == "HumanoidRootPart" then c:Destroy() end end end)
    if oldRoot then pcall(function() oldRoot:Destroy() end); oldRoot = nil end
    if clone then pcall(function() clone:Destroy() end); clone = nil end
    animPlaying = false; _G.invisibleStealEnabled = false
    setToggle("Invisible Steal", false)
    if _G.updateMovementPanelInvisVisual then pcall(_G.updateMovementPanelInvisVisual, false) end
    task.wait(0.2)
    local camera = Workspace.CurrentCamera
    if camera and newChar then
        local h = newChar:FindFirstChildOfClass("Humanoid")
        if h then camera.CameraSubject = h; camera.CameraType = Enum.CameraType.Custom end
    end
end)

function setupDeathListener()
    local ch = player.Character
    if ch then
        local h = ch:FindFirstChildOfClass("Humanoid")
        if h then h.Died:Connect(function() clearErrorOrb(); clearAllGhosts(); lagbackCallCount = 0 end) end
    end
end
setupDeathListener()
player.CharacterAdded:Connect(function() task.wait(0.1); setupDeathListener() end)

-- ANTI-DIE
task.spawn(function()
    _G.AntiDieDisabled = false
    local function setupAntiDie()
        if _G.AntiDieDisabled then return end
        local character = player.Character
        if not character then return end
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        if not humanoid then return end
        if _G.AntiDieConnection then pcall(function() _G.AntiDieConnection:Disconnect() end) end
        _G.AntiDieConnection = humanoid:GetPropertyChangedSignal("Health"):Connect(function()
            if _G.AntiDieDisabled then return end
            if humanoid.Health <= 0 then
                humanoid.Health = humanoid.MaxHealth
            end
        end)
    end
    _G.setupAntiDie = setupAntiDie
    setupAntiDie()
    player.CharacterAdded:Connect(function()
        task.wait(0.5)
        if not _G.AntiDieDisabled then
            setupAntiDie()
        end
    end)
end)

-- Automatic Invis During Steal Loop
task.spawn(function()
    local wasStealingForInvis = false
    local autoEnabledInvis = false
    task.wait(1)
    while task.wait(0.15) do
        if Config.AutoInvisDuringSteal == false then
            wasStealingForInvis = false
            autoEnabledInvis = false
        else
            local isStealing = player:GetAttribute("Stealing")
            if isStealing and not wasStealingForInvis then
                if not _G.invisibleStealEnabled and _G._forceInvisToggle then
                    task.defer(function()
                        if player:GetAttribute("Stealing") and not _G.invisibleStealEnabled then
                            pcall(_G._forceInvisToggle)
                            autoEnabledInvis = true
                        end
                    end)
                end
            end
            if not isStealing and autoEnabledInvis and _G.invisibleStealEnabled and _G._forceInvisToggle then
                task.wait(0.3)
                if not player:GetAttribute("Stealing") then
                    pcall(_G._forceInvisToggle)
                    autoEnabledInvis = false
                end
            end
            wasStealingForInvis = isStealing
        end
    end
end)

-- FLOAT
FloatState={active=false,platform=nil,followConn=nil}
function removeFloatPlatform() if FloatState.followConn then FloatState.followConn:Disconnect(); FloatState.followConn=nil end; if FloatState.platform then FloatState.platform:Destroy(); FloatState.platform=nil end end
function createFloatPlatform()
    removeFloatPlatform(); local c=player.Character; local hrp=c and c:FindFirstChild("HumanoidRootPart"); if not hrp then return end
    local p=Instance.new("Part"); p.Size=Vector3.new(7,1,7); p.Anchored=true; p.CanCollide=true; p.CanTouch=false; p.CanQuery=false
    p.Transparency=1; p.CastShadow=false; p.CFrame=CFrame.new(hrp.Position-Vector3.new(0,3.35,0)); p.Parent=Workspace; FloatState.platform=p
    FloatState.followConn=RunService.Heartbeat:Connect(function() if not FloatState.active then return end
        local ch=player.Character; local h=ch and ch:FindFirstChild("HumanoidRootPart")
        if h and FloatState.platform then FloatState.platform.CFrame=CFrame.new(h.Position-Vector3.new(0,3.35,0)) end
    end)
end
function setFloat(on)
    -- Never allow Float while carrying a stolen brainrot
    if on and player:GetAttribute("Stealing") then
        if ShowNotification then pcall(ShowNotification, "FLOAT", "Blocked while carrying a brainrot") end
        if setToggle then pcall(setToggle, "Float", false) end -- keep the UI toggle in sync
        return
    end
    FloatState.active=on; Config.Float=on; saveConfig(); if on then createFloatPlatform() else removeFloatPlatform() end
end
_G.toggleFloat=function() setFloat(not FloatState.active) end

-- WALKSPEED (CFrame Bypass)
WalkSpeedState = {enabled = false, conn = nil, speed = Config.WalkSpeedValue or 16}
function setWalkSpeedEnabled(en)
    WalkSpeedState.enabled = en
    Config.WalkSpeedEnabled = en
    setToggle("WalkSpeed", en)
    saveConfig()
    if WalkSpeedState.conn then WalkSpeedState.conn:Disconnect(); WalkSpeedState.conn = nil end
    if not en then return end
    WalkSpeedState.conn = RunService.Heartbeat:Connect(function(dt)
        local character = player.Character
        if not character then return end
        local humanoid = character:FindFirstChildOfClass("Humanoid")
        local rootPart = character:FindFirstChild("HumanoidRootPart")
        if not humanoid or not rootPart or humanoid.Health <= 0 then return end
        if humanoid.MoveDirection.Magnitude > 0 and WalkSpeedState.speed > humanoid.WalkSpeed then
            local extraSpeed = WalkSpeedState.speed - humanoid.WalkSpeed
            rootPart.CFrame = rootPart.CFrame + (humanoid.MoveDirection * extraSpeed * dt)
        end
    end)
end
function setWalkSpeedValue(v)
    v = math.clamp(math.floor(v + 0.5), 15, 29)
    WalkSpeedState.speed = v
    Config.WalkSpeedValue = v
    saveConfig()
    return v
end
_G.setWalkSpeedEnabled = setWalkSpeedEnabled
_G.setWalkSpeedValue = setWalkSpeedValue

-- CARPET SPEED
CarpetState={enabled=false,conn=nil}
SPEED_BOOST_TOOL_NAMES = {
    ["Flying Carpet"]=true, ["Carpet"]=true, ["Cloud"]=true,
    ["Witch's Broom"]=true, ["Cupid's Wings"]=true, ["Santa's Sleigh"]=true,
    ["Magic Carpet"]=true, ["Waverider"]=true,
}
function isSpeedBoostTool(tool)
    if not tool or not tool:IsA("Tool") then return false end
    if SPEED_BOOST_TOOL_NAMES[tool.Name] then return true end
    local preferred = Config.TpSettings and Config.TpSettings.Tool
    return preferred ~= nil and tool.Name == preferred
end
function getEquippedTool(char)
    if not char then return nil end
    return char:FindFirstChildWhichIsA("Tool")
end
function equipSpeedBoostToolOnce(c, hum)
    if not c or not hum then return nil end
    local preferred = Config.TpSettings.Tool or "Flying Carpet"
    if c:FindFirstChild(preferred) then return preferred end
    local tb = player.Backpack:FindFirstChild(preferred)
    if tb then pcall(function() hum:EquipTool(tb) end); return preferred end
    for name in pairs(SPEED_BOOST_TOOL_NAMES) do
        if c:FindFirstChild(name) then return name end
        local t = player.Backpack:FindFirstChild(name)
        if t then pcall(function() hum:EquipTool(t) end); return name end
    end
    return nil
end
function isCarpetSpeedBlocked()
    if _G._isTpMoving then return true end
    if player:GetAttribute("Stealing") then return true end
    return false
end
_G.isCarpetSpeedBlocked = isCarpetSpeedBlocked
function setCarpetSpeed(en)
    if en and isCarpetSpeedBlocked() then return end
    CarpetState.enabled=en
    Config["Carpet Speed"]=en
    setToggle("Carpet Speed",en)
    saveConfig()
    if CarpetState.conn then CarpetState.conn:Disconnect(); CarpetState.conn=nil end
    if not en then return end
    local c=player.Character
    local hum=c and c:FindFirstChild("Humanoid")
    if c and hum then equipSpeedBoostToolOnce(c, hum) end
    CarpetState.conn=RunService.Heartbeat:Connect(function()
        local c=player.Character
        if not c then return end
        if isCarpetSpeedBlocked() then
            local hrp=c:FindFirstChild("HumanoidRootPart")
            if hrp then hrp.AssemblyLinearVelocity=Vector3.new(0,hrp.AssemblyLinearVelocity.Y,0) end
            return
        end
        local hum=c:FindFirstChild("Humanoid")
        local hrp=c:FindFirstChild("HumanoidRootPart")
        if not hum or not hrp then return end
        local equipped = getEquippedTool(c)
        if equipped and not isSpeedBoostTool(equipped) then
            hrp.AssemblyLinearVelocity=Vector3.new(0, hrp.AssemblyLinearVelocity.Y, 0)
            setCarpetSpeed(false)
            return
        end
        if equipped and isSpeedBoostTool(equipped) then
            local md=hum.MoveDirection
            if md.Magnitude>0 then
                hrp.AssemblyLinearVelocity=Vector3.new(md.X*140,hrp.AssemblyLinearVelocity.Y,md.Z*140)
            else
                hrp.AssemblyLinearVelocity=Vector3.new(0,hrp.AssemblyLinearVelocity.Y,0)
            end
        end
    end)
end
_G.setCarpetSpeed = setCarpetSpeed

CarpetSpeedToolConn = nil
function stopCarpetSpeedFromToolSwitch(char)
    if not CarpetState.enabled or not char then return end
    local equipped = getEquippedTool(char)
    if equipped and not isSpeedBoostTool(equipped) then
        local hrp = char:FindFirstChild("HumanoidRootPart")
        if hrp then hrp.AssemblyLinearVelocity = Vector3.new(0, hrp.AssemblyLinearVelocity.Y, 0) end
        setCarpetSpeed(false)
    end
end
function bindCarpetSpeedToolWatch(char)
    if CarpetSpeedToolConn then
        pcall(function() CarpetSpeedToolConn:Disconnect() end)
        CarpetSpeedToolConn = nil
    end
    if not char then return end
    CarpetSpeedToolConn = char.ChildAdded:Connect(function(child)
        if child:IsA("Tool") then
            task.defer(function() stopCarpetSpeedFromToolSwitch(char) end)
        end
    end)
end
player.CharacterAdded:Connect(function(char)
    bindCarpetSpeedToolWatch(char)
    if CarpetState.enabled then
        task.defer(function()
            local hum = char:WaitForChild("Humanoid", 10)
            if hum and CarpetState.enabled then equipSpeedBoostToolOnce(char, hum) end
        end)
    end
end)
if player.Character then bindCarpetSpeedToolWatch(player.Character) end

-- INFINITE JUMP (0.1s cooldown, 55 velocity)
InfJumpState = {enabled = false, conn = nil}

function setInfiniteJump(en)
    InfJumpState.enabled = en
    Config.InfiniteJump = en
    Config.TpSettings.InfiniteJump = en
    setToggle("Infinite Jump", en)
    saveConfig()

    if InfJumpState.conn then
        InfJumpState.conn:Disconnect()
        InfJumpState.conn = nil
    end
    if not en then return end

    InfJumpState.conn = RunService.Heartbeat:Connect(function()
        local c = player.Character
        if not c then return end
        local hrp = c:FindFirstChild("HumanoidRootPart")
        local hum = c:FindFirstChild("Humanoid")
        if not hrp or not hum or hum.Health <= 0 then return end

        -- Lógica do seu código original: Verifica se o botão de pulo está pressionado
        -- Adicionei "or hum.Jump" para detectar o botão virtual no celular!
        local isPressingJump = UIS:IsKeyDown(Enum.KeyCode.Space) or hum.Jump

        if isPressingJump then
            -- Removi o "cooldown" e a trava de tempo. Agora, enquanto você segurar,
            -- ele vai aplicar a velocidade 55 constantemente, fazendo ele voar pra cima sem parar.
            hrp.AssemblyLinearVelocity = Vector3.new(hrp.AssemblyLinearVelocity.X, 55, hrp.AssemblyLinearVelocity.Z)
        end
        -- Se soltar o botão, a condição "isPressingJump" vira false, o código não executa essa linha,
        -- e a gravidade do jogo puxa ele de volta para o chão normalmente.
    end)
end

-- ANTI RAGDOLL
-- Anti-Ragdoll System (comprehensive with knockback suppression)
do
    local antiRagdollConnections = {}
    local antiRagdollCharacter, antiRagdollHumanoid, antiRagdollRootPart, antiRagdollAnimator
    local lastVelocity = Vector3.new(0, 0, 0)
    local velocityChangeThreshold = 40
    local velocityMagnitudeThreshold = 25
    local maxVelocity = 15

    local function isFlyingCarpetActive()
        if not antiRagdollCharacter then return false end
        local tool = antiRagdollCharacter:FindFirstChildWhichIsA("Tool")
        if not tool then return false end
        local hrp = antiRagdollCharacter:FindFirstChild("HumanoidRootPart")
        if hrp then
            for _, obj in ipairs(hrp:GetChildren()) do
                if obj:IsA("BodyVelocity") or obj:IsA("BodyPosition") or obj:IsA("BodyGyro") then
                    return true
                end
            end
        end
        return false
    end

    local function isRagdolled()
        if not antiRagdollHumanoid then return false end
        local state = antiRagdollHumanoid:GetState()
        return state == Enum.HumanoidStateType.Physics
            or state == Enum.HumanoidStateType.Ragdoll
            or state == Enum.HumanoidStateType.FallingDown
            or state == Enum.HumanoidStateType.GettingUp
    end

    local function enableAntiRagdollControls()
        pcall(function()
            local PlayerModule = player:WaitForChild("PlayerScripts"):WaitForChild("PlayerModule", 10)
            require(PlayerModule):GetControls():Enable()
        end)
    end

    local function cleanupRagdoll()
        if not antiRagdollCharacter then return end
        local carpetEquipped = isFlyingCarpetActive()

        local function processChildren(parent)
            for _, obj in ipairs(parent:GetChildren()) do
                if obj:IsA("BallSocketConstraint") or obj:IsA("NoCollisionConstraint") or obj:IsA("HingeConstraint")
                    or (obj:IsA("Attachment") and (obj.Name == "A" or obj.Name == "B")) then
                    obj:Destroy()
                elseif obj:IsA("BodyVelocity") or obj:IsA("BodyPosition") or obj:IsA("BodyGyro") then
                    if not carpetEquipped then obj:Destroy() end
                elseif obj:IsA("Motor6D") then
                    obj.Enabled = true
                elseif obj:IsA("BasePart") then
                    for _, child in ipairs(obj:GetChildren()) do
                        if child:IsA("BallSocketConstraint") or child:IsA("NoCollisionConstraint") or child:IsA("HingeConstraint") or child:IsA("Motor6D") then
                            if child:IsA("Motor6D") then
                                child.Enabled = true
                            else
                                child:Destroy()
                            end
                        elseif child:IsA("Attachment") and (child.Name == "A" or child.Name == "B") then
                            child:Destroy()
                        end
                    end
                end
            end
        end

        pcall(function() processChildren(antiRagdollCharacter) end)

        if antiRagdollAnimator then
            for _, track in pairs(antiRagdollAnimator:GetPlayingAnimationTracks()) do
                local animName = track.Animation and track.Animation.Name:lower() or ""
                if animName:find("rag") or animName:find("fall") or animName:find("hurt") or animName:find("down") then
                    track:Stop(0)
                end
            end
        end
    end

    local function setupAntiRagdollCharacter(char)
        antiRagdollCharacter = char
        antiRagdollHumanoid = char:WaitForChild("Humanoid", 10)
        antiRagdollRootPart = char:WaitForChild("HumanoidRootPart", 10)
        antiRagdollAnimator = antiRagdollHumanoid and antiRagdollHumanoid:WaitForChild("Animator", 10)
        lastVelocity = Vector3.new(0, 0, 0)
    end

    local function clearAntiRagdollConnections()
        for _, c in pairs(antiRagdollConnections) do
            pcall(function() c:Disconnect() end)
        end
        antiRagdollConnections = {}
    end

    local function setupAntiRagdollConnections()
        clearAntiRagdollConnections()
        if not antiRagdollHumanoid or not antiRagdollRootPart then return end

        table.insert(antiRagdollConnections, antiRagdollHumanoid.StateChanged:Connect(function()
            if (_G.AntiRagdollEnabled or _G.antiKnockbackEnabled) and isRagdolled() then
                if not isFlyingCarpetActive() then
                    antiRagdollHumanoid:ChangeState(Enum.HumanoidStateType.Running)
                end
                cleanupRagdoll()
                Workspace.CurrentCamera.CameraSubject = antiRagdollHumanoid
                enableAntiRagdollControls()
            end
        end))

        pcall(function()
            local impulsePath = _G.Net and _G.Net:RemoteEvent("CombatService/ApplyImpulse")
            if impulsePath then
                table.insert(antiRagdollConnections, impulsePath.OnClientEvent:Connect(function()
                    if (_G.AntiRagdollEnabled or _G.antiKnockbackEnabled) and isRagdolled() then
                        antiRagdollRootPart.AssemblyLinearVelocity = Vector3.new(0, 0, 0)
                    end
                end))
            end
        end)

        table.insert(antiRagdollConnections, antiRagdollCharacter.DescendantAdded:Connect(function()
            if (_G.AntiRagdollEnabled or _G.antiKnockbackEnabled) and isRagdolled() then
                cleanupRagdoll()
            end
        end))

        table.insert(antiRagdollConnections, RunService.Heartbeat:Connect(function()
            if (_G.AntiRagdollEnabled or _G.antiKnockbackEnabled) and isRagdolled() then
                cleanupRagdoll()
                local velocity = antiRagdollRootPart.AssemblyLinearVelocity
                if (velocity - lastVelocity).Magnitude > velocityChangeThreshold
                    and velocity.Magnitude > velocityMagnitudeThreshold then
                    antiRagdollRootPart.AssemblyLinearVelocity = velocity.Unit * math.min(velocity.Magnitude, maxVelocity)
                end
                lastVelocity = velocity
            end
        end))

        enableAntiRagdollControls()
        cleanupRagdoll()
    end

    function startAntiRagdoll()
        _G.AntiRagdollEnabled = true
        _G.antiKnockbackEnabled = true
        Config.AntiRagdoll = true; setToggle("Anti Ragdoll", true); saveConfig()
        if player.Character then
            setupAntiRagdollCharacter(player.Character)
            setupAntiRagdollConnections()
        end
    end

    function stopAntiRagdoll()
        _G.AntiRagdollEnabled = false
        _G.antiKnockbackEnabled = false
        Config.AntiRagdoll = false; setToggle("Anti Ragdoll", false); saveConfig()
        clearAntiRagdollConnections()
    end

    _G.toggleAntiRagdoll = function(enabled)
        if enabled then startAntiRagdoll() else stopAntiRagdoll() end
    end
    _G.enableAntiKnockback = function() startAntiRagdoll() end
    _G.disableAntiKnockback = function() stopAntiRagdoll() end

    player.CharacterAdded:Connect(function(char)
        clearAntiRagdollConnections()
        antiRagdollCharacter = nil; antiRagdollHumanoid = nil; antiRagdollRootPart = nil; antiRagdollAnimator = nil
        local humanoid = char:WaitForChild("Humanoid", 10)
        local rootPart = char:WaitForChild("HumanoidRootPart", 10)
        if not humanoid or not rootPart then return end
        task.wait(0.2)
        setupAntiRagdollCharacter(char)
        if _G.AntiRagdollEnabled or _G.antiKnockbackEnabled then
            setupAntiRagdollConnections()
        end
    end)

    if player.Character then
        setupAntiRagdollCharacter(player.Character)
        if _G.AntiRagdollEnabled or _G.antiKnockbackEnabled then
            setupAntiRagdollConnections()
        end
    end
end

-- ============================================================
-- DARK THEME: darkens the GAME WORLD via a ColorCorrection effect,
-- with adjustable intensity. The effect is whitelisted in both FPS
-- boosts and the anti-knockback screen-fx cleanup so nothing kills it.
-- (do-block: locals stay off the main chunk's 200-register cap)
do
    local cc = nil
    local function apply()
        if not Config.DarkTheme then
            if cc then pcall(function() cc:Destroy() end); cc = nil end
            return
        end
        if not cc or not cc.Parent then
            cc = Instance.new("ColorCorrectionEffect")
            cc.Name = "MynxxDarkTheme"
            cc.Parent = Lighting
        end
        local i = math.clamp((Config.DarkThemeIntensity or 50) / 100, 0, 1)
        cc.Enabled = true
        cc.Brightness = -0.45 * i
        cc.Contrast = 0.08 * i
        cc.Saturation = -0.12 * i
    end
    function setDarkTheme(on)
        Config.DarkTheme = on and true or false
        saveConfig()
        setToggle("Dark Theme", Config.DarkTheme)
        apply()
    end
    function setDarkThemeIntensity(v)
        Config.DarkThemeIntensity = math.clamp(math.floor(v), 0, 100)
        saveConfig()
        apply()
    end
    -- self-heal: something disabled/removed the effect -> reapply
    task.spawn(function()
        while true do
            task.wait(2)
            if Config.DarkTheme and (not cc or not cc.Parent or not cc.Enabled) then
                pcall(apply)
            end
        end
    end)
end

function applyUnwalk(char, on)
    if not char then return end
    local hum = char:FindFirstChildOfClass("Humanoid")
    local animator = hum and hum:FindFirstChildOfClass("Animator")
    local animate = char:FindFirstChild("Animate")
    if on then
        if animate then animate.Disabled = true end
        if animator then
            local ok, tracks = pcall(function() return animator:GetPlayingAnimationTracks() end)
            if ok and tracks then
                for _, t in ipairs(tracks) do pcall(function() t:Stop(0) end) end
            end
        end
    else
        if animate then animate.Disabled = false end
    end
end

function setUnwalk(on)
    Config.Unwalk = on
    saveConfig()
    setToggle("Unwalk", on)
    if on then
        task.spawn(function()
            local char = player.Character
            for i = 1, 6 do
                if not Config.Unwalk or player.Character ~= char then break end
                applyUnwalk(char, true)
                task.wait(0.3)
            end
        end)
    else
        applyUnwalk(player.Character, false)
    end
end
_G.setUnwalk = setUnwalk

player.CharacterAdded:Connect(function(char)
    task.spawn(function()
        char:WaitForChild("Humanoid", 10)
        task.wait(0.1)
        if Config.Unwalk then
            for i = 1, 6 do
                if not Config.Unwalk or player.Character ~= char then break end
                applyUnwalk(char, true)
                task.wait(0.3)
            end
        end
    end)
end)

function isMyPlot_Instant(plotName)
    local plots = Workspace:FindFirstChild("Plots")
    if not plots then return false end
    local plot = plots:FindFirstChild(plotName)
    if not plot then return false end

    -- Method 1: Synchronizer check 
    local syncSuccess = false
    local isOwner = false
    pcall(function()
        local Packages = ReplicatedStorage:FindFirstChild("Packages")
        local Synchronizer = Packages and require(Packages:FindFirstChild("Synchronizer"))
        if Synchronizer then
            local ch = _G.XenSyncGet(plotName)
            if ch then
                local own = _G.sProp(ch, "Owner")
                if own then
                    syncSuccess = true
                    if (typeof(own) == "Instance" and own == player) or
                       (typeof(own) == "table" and own.UserId == player.UserId) or
                       (typeof(own) == "number" and own == player.UserId) or
                       (typeof(own) == "string" and (own:lower() == player.Name:lower() or own:lower() == player.DisplayName:lower())) then
                        isOwner = true
                    end
                end
            end
        end
    end)
    if syncSuccess then return isOwner end

    -- Method 2: SurfaceGui TextLabel check (Display Name or Username)
    local sign = plot:FindFirstChild("PlotSign")
    if sign then
        local surfaceGui = sign:FindFirstChildWhichIsA("SurfaceGui", true)
        if surfaceGui then
            local label = surfaceGui:FindFirstChildWhichIsA("TextLabel", true)
            if label then
                local text = label.Text:lower()
                if text:find(player.DisplayName:lower(), 1, true) or text:find(player.Name:lower(), 1, true) then
                    return true
                end
            end
        end
    end

    -- Method 3: BillboardGui "YourBase" check
    if sign then
        local yb = sign:FindFirstChild("YourBase")
        if yb and yb:IsA("BillboardGui") and yb.Enabled == true then
            return true
        end
    end

    return false
end
_G.isMyPlot_Instant = isMyPlot_Instant

-- UNLOCK BASE 
function getUnlockHRP()
    local c = LocalPlayer.Character or player.Character
    if not c then return end
    return c:FindFirstChild("HumanoidRootPart") or c:FindFirstChild("UpperTorso")
end

function smartInteract(number)
    local char = LocalPlayer.Character or player.Character
    local hrp = getUnlockHRP()
    if not char or not hrp then return end

    local plots = Workspace:FindFirstChild("Plots")
    if not plots then return end

    local closestPlot, minDistance = nil, 40

    for _, plot in pairs(plots:GetChildren()) do
        local plotPos
        if plot:IsA("Model") then
            plotPos = plot.PrimaryPart and plot.PrimaryPart.Position or plot:GetPivot().Position
        else
            plotPos = plot.Position
        end

        local dist = (hrp.Position - plotPos).Magnitude
        if dist < minDistance then
            closestPlot = plot
            minDistance = dist
        end
    end

    if closestPlot and closestPlot:FindFirstChild("Unlock") then
        local items = {}

        for _, item in pairs(closestPlot.Unlock:GetChildren()) do
            local pos = item:IsA("Model") and item:GetPivot().Position or item.Position
            table.insert(items, {
                Obj = item,
                Y = pos.Y
            })
        end

        table.sort(items, function(a, b)
            return a.Y < b.Y
        end)

        if items[number] then
            for _, pr in pairs(items[number].Obj:GetDescendants()) do
                if pr:IsA("ProximityPrompt") then
                    pcall(function()
                        fireproximityprompt(pr)
                    end)
                end
            end
        end
    end
end

function getCurrentUnlockFloor()
    local hrp = getUnlockHRP()
    if not hrp then return 1 end

    local y = hrp.Position.Y
    if y < 12 then
        return 1
    else
        return 2
    end
end

-- PROXIMITY AP
ProximityAPActive=false
proxAPRing = nil

function createProxAPRing()
    local existing = Workspace:FindFirstChild("XiProxAPRing")
    if existing then existing:Destroy() end
    local r = Instance.new("Part")
    r.Name = "XiProxAPRing"
    r.Shape = Enum.PartType.Cylinder
    r.Anchored = true
    r.CanCollide = false
    r.CanTouch = false
    r.CanQuery = false
    r.CastShadow = false
    r.Material = Enum.Material.Neon
    r.Transparency = 0.7
    r.Color = Color3.fromRGB(255, 40, 40)
    local range = Config.ProximityRange or 15
    r.Size = Vector3.new(0.2, range*2, range*2)
    r.Parent = Workspace
    proxAPRing = r
end

function destroyProxAPRing()
    if proxAPRing then proxAPRing:Destroy(); proxAPRing = nil end
    local e = Workspace:FindFirstChild("XiProxAPRing")
    if e then e:Destroy() end
end

_proxAPRingFrame=0
RunService.Heartbeat:Connect(function()
    if not ProximityAPActive then return end
    _proxAPRingFrame = _proxAPRingFrame + 1
    if _proxAPRingFrame < 2 then return end
    _proxAPRingFrame = 0
    local char = player.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp or not proxAPRing then return end
    local range = Config.ProximityRange or 15
    proxAPRing.Size = Vector3.new(0.2, range * 2, range * 2)
    proxAPRing.CFrame = (hrp.CFrame * CFrame.Angles(0, 0, math.rad(90))) - Vector3.new(0, 2.8, 0)
    -- self-heal: the FPS Boost strips Neon/colors off unprotected parts
    if proxAPRing.Material ~= Enum.Material.Neon then proxAPRing.Material = Enum.Material.Neon end
    if proxAPRing.Transparency > 0.75 then proxAPRing.Transparency = 0.7 end
    proxAPRing.Color = Color3.fromRGB(255, 40, 40)
end)

function setProximityAP(on)
    ProximityAPActive = on
    -- Never save ProximityAP to config â€” always starts OFF
    Config.ProximityAP = false
    setToggle("Proximity", on)
    if on then createProxAPRing() else destroyProxAPRing() end
end

onToggleChanged("Proximity", function(on)
    ProximityAPActive = on
    -- Never save ProximityAP to config â€” always starts OFF
    Config.ProximityAP = false
    if on then createProxAPRing() else destroyProxAPRing() end
end)

task.spawn(function()
    while true do
        task.wait(0.2)
        if ProximityAPActive then
            local mc = player.Character
            local mh = mc and mc:FindFirstChild("HumanoidRootPart")
            if mh then
                for _,p in ipairs(Players:GetPlayers()) do
                    if p ~= player and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
                        if isPlayerBlacklisted(p) then continue end
                        if (p.Character.HumanoidRootPart.Position - mh.Position).Magnitude <= (Config.ProximityRange or 15) then
                            local activeCmds = {}
                            for _,cmd in ipairs(AP_ALL_COMMANDS) do
                                if not apIsOnCooldown(cmd) then
                                    table.insert(activeCmds, cmd)
                                end
                            end
                            for i, cmd in ipairs(activeCmds) do
                                task.spawn(function()
                                    task.wait((i - 1) * 0.01)
                                    runAdminCommand(p, cmd)
                                end)
                            end
                        end
                    end
                end
            end
        end
    end
end)

-- AUTO RESET ON BALLOON / AUTO KICK ON STEAL / CLEAN ERRORS
task.spawn(function() while true do task.wait(1); if not Config.AutoResetBalloon then continue end
    for _,g in ipairs(playerGui:GetDescendants()) do local txt=(g:IsA("TextLabel") or g:IsA("TextButton")) and g.Text
        if txt and string.find(txt,'ran "balloon" on you') then executeReset(true); break end end end end)

task.spawn(function() local kw="you stole"; local hooked=setmetatable({},{__mode="k"})
    local function hookObj(obj) if hooked[obj] then return end; hooked[obj]=true
        if Config.AutoKickOnSteal and string.find(string.lower(tostring(obj.Text or "")),kw,1,true) then kickPlayer(tostring(obj.Text or "")); return end
        obj:GetPropertyChangedSignal("Text"):Connect(function() if Config.AutoKickOnSteal and string.find(string.lower(tostring(obj.Text or "")),kw,1,true) then kickPlayer(tostring(obj.Text or "")) end end)
    end
    local function watchRoot(root) for _,obj in ipairs(root:GetDescendants()) do if obj:IsA("TextLabel") or obj:IsA("TextButton") or obj:IsA("TextBox") then hookObj(obj) end end
        root.DescendantAdded:Connect(function(desc) if desc:IsA("TextLabel") or desc:IsA("TextButton") or desc:IsA("TextBox") then hookObj(desc) end end) end
    for _,g in ipairs(playerGui:GetChildren()) do watchRoot(g) end; playerGui.ChildAdded:Connect(function(g) watchRoot(g) end)
end)

task.spawn(function() local GS=pcall(function() return cloneref(game:GetService("GuiService")) end) and cloneref(game:GetService("GuiService")) or game:GetService("GuiService")
    while true do if Config.CleanErrorGUIs then pcall(function() GS:ClearError() end) end; task.wait(0.1) end end)

-- ============================================================
-- AUTO STEAL / GRAB
-- ============================================================
do
Packages = ReplicatedStorage:WaitForChild("Packages")
Datas = ReplicatedStorage:WaitForChild("Datas")
local Synchronizer = require(Packages:WaitForChild("Synchronizer"))
local AnimalsData = require(Datas:WaitForChild("Animals"))

autoStealEnabled = Config.AutoStealEnabled
if autoStealEnabled == nil then autoStealEnabled = true end
instantStealEnabled = Config.InstantStealEnabled
if instantStealEnabled == nil then instantStealEnabled = true end
stealHighestEnabled = Config.StealHighest
if stealHighestEnabled == nil then stealHighestEnabled = true end
stealPriorityEnabled = Config.StealPriority
stealNearestEnabled = Config.StealNearest
selectedTargetIndex = 1
selectedTargetUID = nil
manuallySelectedUID = nil
currentStealTargetUID = nil
activeProgressTween = nil
instantStealReady = false
instantStealDidInit = false
INSTANT_STEAL_RADIUS = 60
INSTANT_STEAL_COOLDOWN = 0
lastInstantStealTime = 0
PromptMemoryCache = {}
InternalStealCacheData = {}

-- ============================================================
-- AUTO GRAB INTEGRATION 
-- ============================================================
CONFIG = {
    AUTO_STEAL = false,
    RADIUS = 60
}

boxes = {
    {min = Vector3.new(-337.448303, -3.898971, -122.397758), max = Vector3.new(-328.004578, -3.898971, 242.625626)},
    {min = Vector3.new(-327.257660, -3.899109, -122.228622), max = Vector3.new(-320.600891, -3.899109, 242.612259)},
    {min = Vector3.new(-319.783386, -3.898970, -122.227089), max = Vector3.new(-312.908325, -3.898970, 242.585617)},
    {min = Vector3.new(-312.445648, -3.899108, -122.389832), max = Vector3.new(-305.489899, -3.899108, 242.456818)},
    {min = Vector3.new(-305.037048, -3.898970, -122.230743), max = Vector3.new(-293.957489, -3.898970, 242.606873)},
    {min = Vector3.new(-491.448608, -3.898972, -122.253258), max = Vector3.new(-481.811737, -3.898972, 242.615005)},
    {min = Vector3.new(-498.971069, -3.898970, -122.382767), max = Vector3.new(-491.748840, -3.898970, 242.612061)},
    {min = Vector3.new(-506.436737, -3.898972, -122.411476), max = Vector3.new(-499.318542, -3.898972, 242.615982)},
    {min = Vector3.new(-513.783569, -3.898972, -122.223297), max = Vector3.new(-506.801849, -3.898972, 242.627090)},
    {min = Vector3.new(-525.236938, -3.898972, -122.409813), max = Vector3.new(-514.265015, -3.898972, 242.608932)},
}

trackedPrompts = {}
lastFire = {}

SAFE_POLL_RATE = 0.05
SAFE_POLL_OVERRIDE_UNTIL = 0

function _G.getSafePollRate()
    if os.clock() < SAFE_POLL_OVERRIDE_UNTIL then
        return 0.27
    end
    return SAFE_POLL_RATE
end

function _G.triggerSafePollBoost()
    SAFE_POLL_OVERRIDE_UNTIL = os.clock() + 3
end

FIRE_DEBOUNCE = 0.12
FIRE_BURST = 4
ENABLE_BURST = 35
ENABLE_DEBOUNCE = 0.00
ENABLE_COOLDOWN = 0.08
lastEnableFire = {}

function getHRP()
    local char = LocalPlayer.Character
    return char and char:FindFirstChild("HumanoidRootPart")
end

function getBoxIndex(pos)
    for i,b in ipairs(boxes) do
        if pos.X >= math.min(b.min.X,b.max.X) and pos.X <= math.max(b.min.X,b.max.X)
        and pos.Z >= math.min(b.min.Z,b.max.Z) and pos.Z <= math.max(b.min.Z,b.max.Z) then
            return i
        end
    end
end

function getPromptPosition(prompt)
    local p = prompt.Parent
    if not p then return end
    if p:IsA("Attachment") and p.Parent then p = p.Parent end
    if p:IsA("BasePart") then return p.Position elseif p:IsA("Model") then return p:GetPivot().Position end
end

function promptMatchesSelectedPet(prompt)
    if not SharedState then return false end
    local selected = SharedState.SelectedPetData
    if not selected then return false end
    local model = prompt:FindFirstAncestorOfClass("Model")
    if not model then return false end
    if selected.slot then
        local slotAncestor = prompt:FindFirstAncestor(selected.slot)
        if slotAncestor or model.Name == selected.slot or (model.Parent and model.Parent.Name == selected.slot) then return true end
    end
    local name = selected.name or selected.petName
    if name then
        local wantedName = string.lower(name)
        local current = model
        while current do
            if current.Name and string.lower(current.Name) == wantedName then return true end
            current = current.Parent
        end
    end
    return false
end

function isPromptAvailable(prompt, hrpPos)
    if not autoStealEnabled or not instantStealEnabled then return false end
    if not prompt or not prompt.Parent or not prompt.Enabled then return false end
    local pos = getPromptPosition(prompt)
    if not pos then return false end
    local plot = prompt:FindFirstAncestorOfClass("Model")
    if plot then
        local plots = workspace:FindFirstChild("Plots")
        if plots then
            local parentPlot = prompt:FindFirstAncestorWhichIsA("Model")
            while parentPlot and parentPlot.Parent ~= plots do parentPlot = parentPlot.Parent end
            if parentPlot then
                local sign = parentPlot:FindFirstChild("PlotSign")
                if sign then
                    local gui = sign:FindFirstChildWhichIsA("SurfaceGui", true)
                    local label = gui and gui:FindFirstChildWhichIsA("TextLabel", true)
                    if label then
                        local txt = label.Text:lower()
                        if txt:find(game.Players.LocalPlayer.Name:lower(), 1, true) or txt:find(game.Players.LocalPlayer.DisplayName:lower(), 1, true) then return false end
                    end
                end
            end
        end
    end
    if _G.NEAREST_INSTANT_MODE == true then
        -- Box checks removed to allow nearest to work globally based on radius
    end
    if not (_G.NEAREST_INSTANT_MODE == true) then
        if not promptMatchesSelectedPet(prompt) then return false end
    end
    local configuredRadius = Config.AutoGrabRadius or 60
    return (pos - hrpPos).Magnitude <= configuredRadius
end

function canFire(prompt, debounce)
    local t = os.clock()
    local last = lastFire[prompt]
    if last and (t - last) < debounce then return false end
    lastFire[prompt] = t
    return true
end

function firePrompt(prompt, burst, debounce)
    if not prompt or not prompt.Parent or not prompt.Enabled or not canFire(prompt, debounce) then return end
    for i = 1, burst do pcall(function() fireproximityprompt(prompt, 0) end) end
end

function trackPrompt(prompt)
    if trackedPrompts[prompt] then return end
    trackedPrompts[prompt] = true
    local function tryInstantEnableFire()
        if not autoStealEnabled or not instantStealEnabled then return end
        local hrp = getHRP()
        if hrp and isPromptAvailable(prompt, hrp.Position) then
            CONFIG.AUTO_STEAL = true
            local now = os.clock()
            local le = lastEnableFire[prompt]
            if not le or (now - le) >= ENABLE_COOLDOWN then
                lastEnableFire[prompt] = now
                firePrompt(prompt, ENABLE_BURST, ENABLE_DEBOUNCE)
            end
        end
    end
    task.defer(tryInstantEnableFire)
    pcall(function() prompt:GetPropertyChangedSignal("Enabled"):Connect(function() if prompt.Enabled then tryInstantEnableFire() end end) end)
    prompt.AncestryChanged:Connect(function() if not prompt:IsDescendantOf(workspace) then trackedPrompts[prompt] = nil; lastFire[prompt] = nil; lastEnableFire[prompt] = nil end end)
end

function scanBrainrotPrompts()
    local plots = workspace:FindFirstChild("Plots")
    if plots then
        for _, plot in ipairs(plots:GetChildren()) do
            local podiums = plot:FindFirstChild("AnimalPodiums")
            if podiums then
                for _, obj in ipairs(podiums:GetDescendants()) do
                    if obj:IsA("ProximityPrompt") then trackPrompt(obj) end
                end
            end
        end
    end
end

scanBrainrotPrompts()
workspace.DescendantAdded:Connect(function(obj)
    if obj:IsA("ProximityPrompt") and obj:FindFirstAncestor("AnimalPodiums") then trackPrompt(obj) end
end)

task.spawn(function()
    while task.wait(_G.getSafePollRate()) do
        _G.NEAREST_INSTANT_MODE = (stealNearestEnabled == true)
        if autoStealEnabled and instantStealEnabled then
            local hrp = getHRP()
            if not hrp then CONFIG.AUTO_STEAL = false; continue end
            local myPos = hrp.Position
            local anyAvailable = false
            for prompt in pairs(trackedPrompts) do
                if isPromptAvailable(prompt, myPos) then
                    anyAvailable = true
                    if CONFIG.AUTO_STEAL then firePrompt(prompt, FIRE_BURST, FIRE_DEBOUNCE) end
                end
            end
            CONFIG.AUTO_STEAL = anyAvailable
        else
            CONFIG.AUTO_STEAL = false
        end
    end
end)

function isMyBaseAnimal(animalData)
    if not animalData or not animalData.plot then return false end
    return isMyPlot_Instant(animalData.plot)
end

function get_all_pets()
    local out = {}
    local cache = SharedState.AllAnimalsCache
    local minGenStr = Config.StealNearest and Config.TpSettings.MinGenForGrab or Config.TpSettings.MinGenForTp
    local minGen = parseMinGen(minGenStr)
    local myName = LocalPlayer.Name
    local myDisplay = LocalPlayer.DisplayName
    local prioSet = {}
    for _, p in ipairs(priorityList) do prioSet[p:lower()] = true end

    if cache and type(cache) == "table" then
        for _, a in ipairs(cache) do
            if a and (a.genValue or 0) >= 1 and a.owner ~= myName and a.owner ~= myDisplay then
                local isBrainrot = ((tonumber(a.genValue) or 0) >= 10000000)
                local isPriority = (a.name and prioSet[a.name:lower()]) or (a.index and prioSet[a.index:lower()])
                if isBrainrot and a.name then SharedState.BrainrotNames[a.name:lower()] = true end
                if minGen > 0 and (tonumber(a.genValue) or 0) < minGen then
                    if not isBrainrot and not isPriority then continue end
                end
                table.insert(out, {
                    name = a.name,
                    petName = a.name,
                    mpsText = a.genText,
                    mpsValue = tonumber(a.genValue) or 0,
                    petValue = a.petValue or 0,
                    owner = a.owner,
                    plot = a.plot,
                    slot = a.slot,
                    uid = a.uid,
                    mutation = a.mutation,
                    animalData = a,
                })
            end
        end
    end

    -- Ordena por mpsValue (maior primeiro)
    table.sort(out, function(a, b)
        return (a.mpsValue or 0) > (b.mpsValue or 0)
    end)
    return out
end

function get_all_pets_by_value()
    local out = {}
    local cache = SharedState.AllAnimalsCache
    local minGenStr = Config.StealNearest and Config.TpSettings.MinGenForGrab or Config.TpSettings.MinGenForTp
    local minGen = parseMinGen(minGenStr)
    local myName = LocalPlayer.Name
    local myDisplay = LocalPlayer.DisplayName
    if cache and type(cache) == "table" then
        for _, a in ipairs(cache) do
            if a and (a.genValue or 0) >= 1 and a.owner ~= myName and a.owner ~= myDisplay then
                if minGen > 0 and (tonumber(a.genValue) or 0) < minGen then continue end
                table.insert(out, {
                    name = a.name,
                    petName = a.name,
                    mpsText = a.genText,
                    mpsValue = a.genValue,
                    petValue = a.petValue or 0,
                    owner = a.owner,
                    plot = a.plot,
                    slot = a.slot,
                    uid = a.uid,
                    mutation = a.mutation,
                    animalData = a,
                })
            end
        end
    end
    table.sort(out, function(a, b) return (a.petValue or 0) > (b.petValue or 0) end)
    return out
end

function findProximityPromptForAnimal(animalData)
    if not animalData then return nil end
    local cp = PromptMemoryCache[animalData.uid]
    if cp and cp.Parent then return cp end
    local plot = Workspace.Plots:FindFirstChild(animalData.plot)
    if not plot then return nil end
    local podiums = plot:FindFirstChild("AnimalPodiums")
    if not podiums then return nil end
    local ch = _G.XenSyncGet(plot.Name)
    if not ch then
        local podium = podiums:FindFirstChild(animalData.slot)
        if podium then
            local base = podium:FindFirstChild("Base")
            local spawn = base and base:FindFirstChild("Spawn")
            if spawn then
                local attach = spawn:FindFirstChild("PromptAttachment")
                if attach then
                    for _, p in ipairs(attach:GetChildren()) do
                        if p:IsA("ProximityPrompt") then
                            PromptMemoryCache[animalData.uid] = p
                            return p
                        end
                    end
                end
            end
        end
        return nil
    end
    local al = ch:Get("AnimalList")
    if not al then return nil end
    local brainrotName = (animalData.name and animalData.name:lower()) or ""
    local targetSlot = animalData.slot
    local foundPodium = nil
    for slot, ad in pairs(al) do
        if (type(ad) == "table") and (tostring(slot) == targetSlot) then
            local aName, aInfo = ad.Index, AnimalsData[ad.Index]
            if aInfo and ((aInfo.DisplayName or aName):lower() == brainrotName) then
                foundPodium = podiums:FindFirstChild(tostring(slot))
                break
            end
        end
    end
    if not foundPodium then foundPodium = podiums:FindFirstChild(animalData.slot) end
    if foundPodium then
        local base = foundPodium:FindFirstChild("Base")
        local spawn = base and base:FindFirstChild("Spawn")
        if spawn then
            local attach = spawn:FindFirstChild("PromptAttachment")
            if attach then
                for _, p in ipairs(attach:GetChildren()) do
                    if p:IsA("ProximityPrompt") and p.Enabled and (p.ActionText:lower():find("steal") or p.ActionText == "") then
                        PromptMemoryCache[animalData.uid] = p
                        return p
                    end
                end
            end
            local startPos = spawn.Position
            local slotX, slotZ = startPos.X, startPos.Z
            local nearestPrompt = nil
            local minDist = math.huge
            for _, desc in pairs(plot:GetDescendants()) do
                if desc:IsA("ProximityPrompt") and desc.Enabled and (desc.ActionText:lower():find("steal") or desc.ActionText == "") then
                    local part = desc.Parent
                    local promptPos = nil
                    if part and part:IsA("BasePart") then
                        promptPos = part.Position
                    elseif part and part:IsA("Attachment") and part.Parent and part.Parent:IsA("BasePart") then
                        promptPos = part.Parent.Position
                    end
                    if promptPos then
                        local checkStartY = startPos.Y
                        if brainrotName:find("la secret combinasion") then checkStartY = startPos.Y - 5 end
                        local horizontalDist = math.sqrt(((promptPos.X - slotX) ^ 2) + ((promptPos.Z - slotZ) ^ 2))
                        if (horizontalDist < 5) and (promptPos.Y > checkStartY) then
                            local yDist = promptPos.Y - checkStartY
                            if yDist < minDist then
                                minDist = yDist
                                nearestPrompt = desc
                            end
                        end
                    end
                end
            end
            if nearestPrompt then
                PromptMemoryCache[animalData.uid] = nearestPrompt
                return nearestPrompt
            end
        end
    end
    return nil
end

-- executeInstantSteal logic removed

function setStealMode(mode)
    manuallySelectedUID = nil
    stealHighestEnabled = (mode == "Highest")
    stealPriorityEnabled = (mode == "Priority")
    stealNearestEnabled = (mode == "Nearest")
    Config.StealHighest = stealHighestEnabled
    Config.StealPriority = stealPriorityEnabled
    Config.StealNearest = stealNearestEnabled
    Config.StealMode = mode
    saveConfig()
    setToggle("Steal Highest", stealHighestEnabled)
    setToggle("Steal Priority", stealPriorityEnabled)
    setToggle("Steal Nearest", stealNearestEnabled)
    -- Bridge to Mynxx Clone-TP engine
    if _G.MynxxStealMode then pcall(_G.MynxxStealMode, mode) end
end

-- Create the Bottom Steal HUD matching 
hudGui = playerGui:FindFirstChild("AutoStealCurrentTargetHUD")
if hudGui then hudGui:Destroy() end
hudGui = Instance.new("ScreenGui")
hudGui.Name = "AutoStealCurrentTargetHUD"
hudGui.ResetOnSpawn = false
hudGui.IgnoreGuiInset = true
hudGui.DisplayOrder = 998
hudGui.ZIndexBehavior = Enum.ZIndexBehavior.Sibling
hudGui.Parent = playerGui
STEALBAR = {
    PANEL = Color3.fromRGB(12, 12, 14),
    TEXT = Color3.fromRGB(255, 255, 255),
    STROKE = Color3.fromRGB(52, 52, 58),
    GLOW = Color3.fromRGB(80, 80, 88),
    TRACK = Color3.fromRGB(38, 38, 42),
    TRACK2 = Color3.fromRGB(26, 26, 30),
    FILL1 = Color3.fromRGB(255, 255, 255),
    FILL2 = Color3.fromRGB(200, 200, 208),
}

mobileScale = UIS.TouchEnabled and 0.6 or 1
targetHud = Instance.new("Frame", hudGui)
targetHud.Name = "CurrentTargetHUD"
targetHud.AnchorPoint = Vector2.new(0.5, 1)
targetHud.Size = UDim2.new(0, 230 * mobileScale, 0, 46 * mobileScale)
targetHud.Position = UDim2.new(0.5, 0, 1, -155)    -- sobe mais 20 pixels
targetHud.BackgroundColor3 = STEALBAR.PANEL
targetHud.BackgroundTransparency = UITransparency.Hud
targetHud.BorderSizePixel = 0
targetHud.ZIndex = 70
Instance.new("UICorner", targetHud).CornerRadius = UDim.new(0, math.floor(12 * mobileScale))

hudStroke = Instance.new("UIStroke", targetHud)
hudStroke.Color = STEALBAR.STROKE
hudStroke.Thickness = 1
hudStroke.Transparency = 0.35

hudGlow = Instance.new("UIStroke", targetHud)
hudGlow.Color = STEALBAR.GLOW
hudGlow.Thickness = 3
hudGlow.Transparency = 0.84
hudGlow.ApplyStrokeMode = Enum.ApplyStrokeMode.Border

hudShadow = Instance.new("ImageLabel", targetHud)
hudShadow.Name = "Shadow"
hudShadow.AnchorPoint = Vector2.new(0.5, 0.5)
hudShadow.Position = UDim2.new(0.5, 0, 0.5, 1)
hudShadow.Size = UDim2.new(1, 20, 1, 20)
hudShadow.BackgroundTransparency = 1
hudShadow.Image = "rbxassetid://6014261993"
hudShadow.ImageColor3 = Color3.new(0, 0, 0)
hudShadow.ImageTransparency = 0.72
hudShadow.ScaleType = Enum.ScaleType.Slice
hudShadow.SliceCenter = Rect.new(49, 49, 450, 450)
hudShadow.ZIndex = 69

hudName = Instance.new("TextLabel", targetHud)
hudName.Name = "TargetName"
hudName.Size = UDim2.new(1, -12, 0, 13 * mobileScale)
hudName.Position = UDim2.fromOffset(6 * mobileScale, 3 * mobileScale)
hudName.BackgroundTransparency = 1
hudName.Font = Enum.Font.GothamBold
hudName.TextSize = 11 * mobileScale
hudName.TextColor3 = STEALBAR.TEXT
hudName.TextXAlignment = Enum.TextXAlignment.Center
hudName.TextTruncate = Enum.TextTruncate.AtEnd
hudName.ZIndex = 72
hudName.Text = "No target"

hudProgressBg = Instance.new("Frame", targetHud)
hudProgressBg.Name = "ProgressBg"
hudProgressBg.Size = UDim2.new(1, -10 * mobileScale, 0, 18 * mobileScale)
hudProgressBg.Position = UDim2.fromOffset(5 * mobileScale, 18 * mobileScale)
hudProgressBg.BackgroundColor3 = STEALBAR.TRACK
hudProgressBg.BorderSizePixel = 0
hudProgressBg.ZIndex = 72
Instance.new("UICorner", hudProgressBg).CornerRadius = UDim.new(0, math.floor(8 * mobileScale))

hudProgressBgStroke = Instance.new("UIStroke", hudProgressBg)
hudProgressBgStroke.Color = STEALBAR.STROKE
hudProgressBgStroke.Thickness = 1
hudProgressBgStroke.Transparency = 0.55

hudInnerTrack = Instance.new("Frame", hudProgressBg)
hudInnerTrack.Name = "InnerTrack"
hudInnerTrack.Size = UDim2.new(1, -2, 1, -2)
hudInnerTrack.Position = UDim2.fromOffset(1, 1)
hudInnerTrack.BackgroundColor3 = STEALBAR.TRACK2
hudInnerTrack.BackgroundTransparency = 0.15
hudInnerTrack.BorderSizePixel = 0
hudInnerTrack.ZIndex = 72
Instance.new("UICorner", hudInnerTrack).CornerRadius = UDim.new(0, math.floor(7 * mobileScale))

hudProgressFill = Instance.new("Frame", hudProgressBg)
hudProgressFill.Name = "ProgressFill"
hudProgressFill.Size = UDim2.new(0, 0, 1, 0)
hudProgressFill.BackgroundColor3 = STEALBAR.FILL1
hudProgressFill.BorderSizePixel = 0
hudProgressFill.ZIndex = 73
Instance.new("UICorner", hudProgressFill).CornerRadius = UDim.new(0, math.floor(8 * mobileScale))

hudProgressFillGradient = Instance.new("UIGradient", hudProgressFill)
hudProgressFillGradient.Color = ColorSequence.new({
    ColorSequenceKeypoint.new(0, STEALBAR.FILL1),
    ColorSequenceKeypoint.new(1, STEALBAR.FILL2),
})

hudProgressFillStroke = Instance.new("UIStroke", hudProgressFill)
hudProgressFillStroke.Color = Color3.fromRGB(255, 255, 255)
hudProgressFillStroke.Thickness = 1
hudProgressFillStroke.Transparency = 0.45

hudPercent = Instance.new("TextLabel", hudProgressBg)
hudPercent.Name = "Percent"
hudPercent.Size = UDim2.new(1, 0, 1, 0)
hudPercent.BackgroundTransparency = 1
hudPercent.Font = Enum.Font.GothamBold
hudPercent.TextSize = 12 * mobileScale
hudPercent.TextColor3 = STEALBAR.TEXT
hudPercent.TextStrokeTransparency = 0.7
hudPercent.TextXAlignment = Enum.TextXAlignment.Center
hudPercent.ZIndex = 74
hudPercent.Text = "0%"

-- Unlock floor buttons (1 / 2 / 3) above steal bar
-- Unlock floor buttons (1 / 2 / 3) below steal bar
unlockRow = Instance.new("Frame", hudGui)
unlockRow.Name = "UnlockFloorButtons"
unlockRow.AnchorPoint = Vector2.new(0.5, 0)  -- ancora no topo
unlockRow.Size = UDim2.new(0, 230 * mobileScale, 0, 28 * mobileScale)
-- Posiciona imediatamente abaixo da barra (targetHud)
unlockRow.Position = UDim2.new(0.5, 0, 1, -212 + (46 * mobileScale) + (14 * mobileScale))
unlockRow.BackgroundTransparency = 1
unlockRow.ZIndex = 70

unlockLayout = Instance.new("UIListLayout", unlockRow)
unlockLayout.FillDirection = Enum.FillDirection.Horizontal
unlockLayout.HorizontalAlignment = Enum.HorizontalAlignment.Center
unlockLayout.VerticalAlignment = Enum.VerticalAlignment.Center
unlockLayout.Padding = UDim.new(0, 6)

for floorNum = 1, 3 do
    local unlockBtn = Instance.new("TextButton", unlockRow)
    unlockBtn.Name = "UnlockFloor" .. floorNum
    unlockBtn.Size = UDim2.new(0, 68 * mobileScale, 1, 0)
    unlockBtn.BackgroundColor3 = STEALBAR.PANEL
    unlockBtn.BackgroundTransparency = UITransparency.Button
    unlockBtn.Text = tostring(floorNum)
    unlockBtn.Font = Enum.Font.GothamBlack
    unlockBtn.TextSize = 14 * mobileScale
    unlockBtn.TextColor3 = STEALBAR.TEXT
    unlockBtn.AutoButtonColor = false
    unlockBtn.BorderSizePixel = 0
    unlockBtn.ZIndex = 71
    Instance.new("UICorner", unlockBtn).CornerRadius = UDim.new(0, math.floor(8 * mobileScale))
    local unlockStroke = Instance.new("UIStroke", unlockBtn)
    unlockStroke.Color = STEALBAR.GLOW
    unlockStroke.Thickness = 1
    unlockStroke.Transparency = 0.4
    unlockBtn.MouseButton1Click:Connect(function()
        pcall(smartInteract, floorNum)
    end)
    unlockBtn.MouseEnter:Connect(function()
        unlockBtn.BackgroundColor3 = STEALBAR.TRACK
    end)
    unlockBtn.MouseLeave:Connect(function()
        unlockBtn.BackgroundColor3 = STEALBAR.PANEL
    end)
end

_G.smartInteract = smartInteract

function applySelection(newIndex, pets)
    -- Se for modo Highest, IGNORA o newIndex e encontra o maior valor
    if stealHighestEnabled and pets and #pets > 0 then
        local bestIndex = 1
        local bestValue = pets[1] and pets[1].mpsValue or 0
        for i = 2, #pets do
            local val = pets[i].mpsValue or 0
            if val > bestValue then
                bestValue = val
                bestIndex = i
            end
        end
        newIndex = bestIndex
    end

    if newIndex and (newIndex >= 1) and (newIndex <= #pets) then
        local newUID = pets[newIndex].uid
        if (selectedTargetIndex ~= newIndex) or (selectedTargetUID ~= newUID) or (SharedState.SelectedPetData == nil) then
            selectedTargetIndex = newIndex
            selectedTargetUID = newUID
            SharedState.SelectedPetData = pets[newIndex]
            if SharedState.SelectedPetData then 
                SharedState.LastTargetedPetMpsValue = SharedState.SelectedPetData.mpsValue or 0 
            end
        end
    end
end
-- Update Selection Thread
task.spawn(function()
    while true do
        task.wait(0.1)
        if autoStealEnabled then
            local pets = get_all_pets()
            if #pets > 0 then
                if manuallySelectedUID then
                    local found = false
                    for i, p in ipairs(pets) do if p.uid == manuallySelectedUID then applySelection(i, pets); found = true; break end end
                    if not found then manuallySelectedUID = nil; selectedTargetUID = nil; SharedState.SelectedPetData = nil end
                elseif stealPriorityEnabled then
                    local foundPrioIndex = nil
                    for _, pName in ipairs(priorityList) do
                        local searchName = pName:lower()
                        for i, p in ipairs(pets) do if (p.petName and p.petName:lower() == searchName) or (p.animalData and p.animalData.index and p.animalData.index:lower() == searchName) then foundPrioIndex = i; break end end
                        if foundPrioIndex then break end
                    end
                    applySelection(foundPrioIndex, pets)
                elseif stealNearestEnabled then
                    local hrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
                    if hrp then
                        local bestIndex, bestDist = nil, math.huge
                        for i, p in ipairs(pets) do
                            local targetPart = p.animalData and findAdorneeGlobal(p.animalData)
                            if targetPart and targetPart:IsA("BasePart") then
                                local d = (hrp.Position - targetPart.Position).Magnitude
                                if d < bestDist then bestDist = d; bestIndex = i end
                            end
                        end
                        if bestIndex then
                            applySelection(bestIndex, pets)
                        else
                            SharedState.SelectedPetData = nil
                        end
                    else applySelection(1, pets) end
                else applySelection(1, pets) end
            else SharedState.SelectedPetData = nil end
        else SharedState.SelectedPetData = nil end
    end
end)

-- Heartbeat loop updating bottom HUD labels & executing steals
function _pbFmtVal(num)
    if not num or num == 0 then return "0" end
    if num >= 1e9 then return string.format("%.2fb", num / 1e9) end
    if num >= 1e6 then return string.format("%.2fm", num / 1e6) end
    if num >= 1e3 then return string.format("%.1fk", num / 1e3) end
    return tostring(math.floor(num))
end

-- Ragdoll detection (integrated into steal bar for preshot timing)
RAGBAR_STATES = {
    [Enum.HumanoidStateType.Physics]     = true,
    [Enum.HumanoidStateType.Ragdoll]     = true,
    [Enum.HumanoidStateType.FallingDown] = true,
    [Enum.HumanoidStateType.GettingUp]   = true,
}

function checkRagdollBar()
    local c = LocalPlayer.Character
    local hum = c and c:FindFirstChildOfClass("Humanoid")
    if not hum then return false, nil end
    local endTime = LocalPlayer:GetAttribute("RagdollEndTime")
    local left = endTime and (endTime - workspace:GetServerTimeNow()) or nil
    if RAGBAR_STATES[hum:GetState()] then
        return true, left
    end
    if left and left > 0 then
        return true, left
    end
    return false, nil
end

RAG_RED = Color3.fromRGB(255, 60, 60)
RAG_ORANGE = Color3.fromRGB(255, 160, 60)
ragTotalDuration, ragLastEndTime = nil, nil

RunService.RenderStepped:Connect(function()
    local isRag, ragLeft = checkRagdollBar()
    if isRag then
        hudStroke.Color = RAG_RED
        hudGlow.Color = RAG_RED
        hudProgressFill.BackgroundColor3 = RAG_RED
        if ragLeft and ragLeft > 0 then
            -- lock in the total duration at the start of this ragdoll so the
            -- bar starts full and drains down to 0
            local endTime = LocalPlayer:GetAttribute("RagdollEndTime")
            if endTime ~= ragLastEndTime then
                ragLastEndTime = endTime
                ragTotalDuration = ragLeft
            end
            local p = math.clamp(ragLeft / math.max(ragTotalDuration or ragLeft, 0.1), 0, 1)
            hudName.Text = string.format("RAGDOLL - %.1fs", ragLeft)
            hudProgressFill.Size = UDim2.new(p, 0, 1, 0)
            hudPercent.Text = string.format("%.1fs", ragLeft)
            hudPercent.TextColor3 = RAG_ORANGE
        else
            hudName.Text = "RAGDOLL"
            hudProgressFill.Size = UDim2.new(1, 0, 1, 0)
            hudPercent.Text = "..."
            hudPercent.TextColor3 = RAG_ORANGE
        end
        return
    else
        hudStroke.Color = STEALBAR.STROKE
        hudGlow.Color = STEALBAR.GLOW
        hudPercent.TextColor3 = STEALBAR.TEXT
    end

    if not (Config and Config.AutoStealEnabled) then
        hudName.Text = "Disabled"
        hudProgressFill.Size = UDim2.new(0, 0, 1, 0)
        hudPercent.Text = "0%"
        return
    end

    if LocalPlayer:GetAttribute("Stealing") then
        hudProgressFill.Size = UDim2.new(1, 0, 1, 0)
        hudProgressFill.BackgroundColor3 = Color3.fromRGB(100, 220, 130)
        hudPercent.Text = "100%"
        hudName.Text = "Carrying Brainrot!"
        return
    end

    local status = _G.Mynxx_StealStatus or {}
    if status.active then
        local p = math.clamp((tick() - (status.start or 0)) / (status.duration or 1.3), 0, 1)
        hudProgressFill.Size = UDim2.new(p, 0, 1, 0)
        hudPercent.Text = math.floor(p * 100) .. "%"
        hudProgressFill.BackgroundColor3 = (p >= 1) and Color3.fromRGB(100, 220, 130) or STEALBAR.FILL1
        hudName.Text = "Stealing..."
    elseif status.target then
        hudProgressFill.Size = UDim2.new(0, 0, 1, 0)
        hudPercent.Text = "0%"
        hudProgressFill.BackgroundColor3 = STEALBAR.FILL1
        local vs = _pbFmtVal(status.target.mps or status.target.value)
        hudName.Text = (status.target.name or "Brainrot") .. ((vs ~= "0") and (" - $" .. vs) or "")
    else
        hudProgressFill.Size = UDim2.new(0, 0, 1, 0)
        hudPercent.Text = "0%"
        hudProgressFill.BackgroundColor3 = STEALBAR.FILL1
        hudName.Text = "No items"
    end
end)

end


-- ============================================================
-- PRIORITY ALERT (Mynxx Styled 3D Viewer with Sorting Queue)
-- ============================================================
function ShowPriorityAlertImpl(brainrotName, genText, mutation, ownerUsername, headerText)
    -- Toggle "Brainrot Notification" (ESP tab). nil = old config = enabled.
    if Config and Config.BrainrotAlert == false then return end
    local normalizedMutation = mutation and mutation:gsub("%s+", ""):lower() or ""
    
    local mutationColors = {
        ["gold"] = Color3.fromRGB(255, 222, 89),
        ["diamond"] = Color3.fromRGB(37, 196, 254),
        ["bloodrot"] = Color3.fromRGB(145, 0, 27),
        ["rainbow"] = Color3.fromRGB(255, 0, 251),
        ["candy"] = Color3.fromRGB(255, 70, 246),
        ["lava"] = Color3.fromRGB(255, 149, 0),
        ["galaxy"] = Color3.fromRGB(170, 60, 255),
        ["yinyang"] = Color3.fromRGB(255, 255, 255),
        ["radioactive"] = Color3.fromRGB(104, 245, 0),
        ["cursed"] = Color3.fromRGB(245, 56, 56),
        ["divine"] = Color3.fromRGB(255, 209, 59),
        ["cyber"] = Color3.fromRGB(121, 219, 255),
        ["normal"] = Color3.fromRGB(160, 160, 160),
        ["none"] = Color3.fromRGB(160, 160, 160)
    }
    
    local ExploitGui = (gethui and gethui()) or game:GetService("CoreGui")
    local accent = mutationColors[normalizedMutation] or Color3.fromRGB(120, 205, 255)
    
    local subText = ""
    if normalizedMutation ~= "" and normalizedMutation ~= "none" then
        subText = mutation:upper()
    else
        subText = "NORMAL"
    end
    
    local existing = ExploitGui:FindFirstChild("XiPriorityAlertTest")
    if existing then existing:Destroy() end
    
    if Config.PrioritySoundAlert and Config.PrioritySoundID and Config.PrioritySoundID ~= "" then
        pcall(function()
            local sid = Config.PrioritySoundID:match("%d+")
            if sid then
                local snd = Instance.new("Sound")
                snd.SoundId = "rbxassetid://" .. sid
                snd.Volume = 1
                snd.Parent = game:GetService("SoundService")
                snd:Play()
                game:GetService("Debris"):AddItem(snd, 5)
            end
        end)
    end
    
    -- NOTE: no addLazyUI here. It set Enabled=false and relied on the ONE-SHOT
    -- 4s lazy-UI sweep to re-enable: any alert fired AFTER that sweep stayed
    -- disabled forever, and one fired before it was mostly eaten (the alert
    -- self-destroys shortly after creation). That's why the startup "best
    -- brainrot" notif only showed up sometimes.
    local SHOW_TIME = 5.5
    local alertGui = Instance.new("ScreenGui")
    alertGui.Enabled = true
    alertGui.Name = "XiPriorityAlertTest"
    alertGui.ResetOnSpawn = false
    alertGui.DisplayOrder = 999
    alertGui.Parent = ExploitGui
    
    -- Glassy dark card (always dark: it overlays the 3D world, not the hub)
    local alertFrame = Instance.new("Frame")
    alertFrame.Size = UDim2.new(0, 340, 0, 92)
    alertFrame.Position = UDim2.new(0.5, 0, 0, -130) -- Start off-screen top
    alertFrame.AnchorPoint = Vector2.new(0.5, 0)
    alertFrame.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    alertFrame.BackgroundTransparency = 0.04
    alertFrame.BorderSizePixel = 0
    alertFrame.ClipsDescendants = true
    alertFrame.Parent = registerScreenGui(alertGui)
    
    local scaleObj = Instance.new("UIScale", alertFrame)
    scaleObj.Scale = 0.85
    
    local gradient = Instance.new("UIGradient", alertFrame)
    gradient.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(26, 26, 34)),
        ColorSequenceKeypoint.new(0.55, Color3.fromRGB(15, 15, 20)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(10, 10, 14))
    })
    gradient.Rotation = 90
    
    local corner = Instance.new("UICorner", alertFrame)
    corner.CornerRadius = UDim.new(0, 16)
    
    local stroke = Instance.new("UIStroke", alertFrame)
    stroke.Color = accent
    stroke.Thickness = 1.6
    stroke.Transparency = 0.15
    local strokeGrad = Instance.new("UIGradient", stroke)
    strokeGrad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, accent),
        ColorSequenceKeypoint.new(0.5, Color3.fromRGB(255, 255, 255)),
        ColorSequenceKeypoint.new(1, accent)
    })
    strokeGrad.Rotation = 25
    
    -- soft colored halo behind the card content (top glow)
    local halo = Instance.new("Frame", alertFrame)
    halo.Size = UDim2.new(1, 0, 0, 46)
    halo.Position = UDim2.new(0, 0, 0, 0)
    halo.BackgroundColor3 = accent
    halo.BackgroundTransparency = 0.92
    halo.BorderSizePixel = 0
    halo.ZIndex = 0
    local haloGrad = Instance.new("UIGradient", halo)
    haloGrad.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 0),
        NumberSequenceKeypoint.new(1, 1)
    })
    haloGrad.Rotation = 90
    
    -- header kicker (what kind of alert this is)
    local headerLabel = Instance.new("TextLabel", alertFrame)
    headerLabel.Size = UDim2.new(1, -120, 0, 12)
    headerLabel.Position = UDim2.new(0, 18, 0, 12)
    headerLabel.BackgroundTransparency = 1
    headerLabel.Text = (headerText or "PRIORITY BRAINROT")
    headerLabel.Font = Enum.Font.GothamBold
    headerLabel.TextSize = 9
    headerLabel.TextColor3 = accent
    headerLabel.TextTransparency = 0.15
    headerLabel.TextXAlignment = Enum.TextXAlignment.Left
    
    -- brainrot name
    local titleLabel = Instance.new("TextLabel", alertFrame)
    titleLabel.Size = UDim2.new(1, -120, 0, 20)
    titleLabel.Position = UDim2.new(0, 18, 0, 25)
    titleLabel.BackgroundTransparency = 1
    titleLabel.Text = brainrotName
    titleLabel.Font = Enum.Font.GothamBlack
    titleLabel.TextSize = 16
    titleLabel.TextColor3 = Color3.fromRGB(255, 255, 255)
    titleLabel.TextXAlignment = Enum.TextXAlignment.Left
    titleLabel.TextTruncate = Enum.TextTruncate.AtEnd
    
    -- generation ($/s) -- the money shot, big and colored
    local genLabel = Instance.new("TextLabel", alertFrame)
    genLabel.Size = UDim2.new(1, -120, 0, 16)
    genLabel.Position = UDim2.new(0, 18, 0, 47)
    genLabel.BackgroundTransparency = 1
    genLabel.Text = genText or ""
    genLabel.Font = Enum.Font.GothamBlack
    genLabel.TextSize = 14
    genLabel.TextColor3 = accent
    genLabel.TextXAlignment = Enum.TextXAlignment.Left
    
    -- owner
    local subLabel = Instance.new("TextLabel", alertFrame)
    subLabel.Size = UDim2.new(1, -120, 0, 12)
    subLabel.Position = UDim2.new(0, 18, 0, 66)
    subLabel.BackgroundTransparency = 1
    subLabel.Text = "Owner: " .. tostring(ownerUsername)
    subLabel.Font = Enum.Font.GothamMedium
    subLabel.TextSize = 10
    subLabel.TextColor3 = Color3.fromRGB(165, 165, 175)
    subLabel.TextXAlignment = Enum.TextXAlignment.Left
    
    -- mutation pill, top-right
    local chip = Instance.new("TextLabel", alertFrame)
    chip.AutomaticSize = Enum.AutomaticSize.X
    chip.Size = UDim2.new(0, 0, 0, 18)
    chip.Position = UDim2.new(1, -12, 0, 12)
    chip.AnchorPoint = Vector2.new(1, 0)
    chip.BackgroundColor3 = accent
    chip.BackgroundTransparency = 0.85
    chip.BorderSizePixel = 0
    chip.Text = subText
    chip.Font = Enum.Font.GothamBold
    chip.TextSize = 9
    chip.TextColor3 = accent
    local chipPad = Instance.new("UIPadding", chip)
    chipPad.PaddingLeft = UDim.new(0, 9)
    chipPad.PaddingRight = UDim.new(0, 9)
    local chipCorner = Instance.new("UICorner", chip)
    chipCorner.CornerRadius = UDim.new(1, 0)
    local chipStroke = Instance.new("UIStroke", chip)
    chipStroke.Color = accent
    chipStroke.Thickness = 1
    chipStroke.Transparency = 0.55
    
    -- countdown bar along the bottom (accent colored, drains over SHOW_TIME)
    local countBar = Instance.new("Frame", alertFrame)
    countBar.Size = UDim2.new(1, -24, 0, 3)
    countBar.Position = UDim2.new(0.5, 0, 1, -5)
    countBar.AnchorPoint = Vector2.new(0.5, 1)
    countBar.BackgroundColor3 = accent
    countBar.BorderSizePixel = 0
    local countCorner = Instance.new("UICorner", countBar)
    countCorner.CornerRadius = UDim.new(1, 0)
    
    -- shine sweep across the card right after it lands
    -- (no Rotation here: ClipsDescendants doesn't clip rotated frames, the
    -- diagonal look comes from the gradient's rotation instead)
    local shine = Instance.new("Frame", alertFrame)
    shine.Size = UDim2.new(0, 80, 1, 0)
    shine.Position = UDim2.new(-0.35, 0, 0, 0)
    shine.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    shine.BackgroundTransparency = 0
    shine.BorderSizePixel = 0
    shine.ZIndex = 5
    local shineGrad = Instance.new("UIGradient", shine)
    shineGrad.Rotation = 15
    shineGrad.Transparency = NumberSequence.new({
        NumberSequenceKeypoint.new(0, 1),
        NumberSequenceKeypoint.new(0.5, 0.82),
        NumberSequenceKeypoint.new(1, 1)
    })
    
    -- ENTRANCE: drop + settle, scale pop, then the shine sweep
    TweenService:Create(alertFrame, TweenInfo.new(0.55, Enum.EasingStyle.Exponential, Enum.EasingDirection.Out), {
        Position = UDim2.new(0.5, 0, 0, 22)
    }):Play()
    TweenService:Create(scaleObj, TweenInfo.new(0.5, Enum.EasingStyle.Back, Enum.EasingDirection.Out), {
        Scale = 1
    }):Play()
    task.delay(0.35, function()
        if shine and shine.Parent then
            TweenService:Create(shine, TweenInfo.new(0.8, Enum.EasingStyle.Quad, Enum.EasingDirection.InOut), {
                Position = UDim2.new(1.3, 0, 0, 0)
            }):Play()
            -- one-shot effect: free the frame + gradient once the sweep is done
            task.delay(0.85, function()
                if shine and shine.Parent then shine:Destroy() end
            end)
        end
    end)
    TweenService:Create(countBar, TweenInfo.new(SHOW_TIME, Enum.EasingStyle.Linear), {
        Size = UDim2.new(0, 0, 0, 3)
    }):Play()
    
    task.delay(SHOW_TIME, function()
        if alertFrame and alertFrame.Parent then
            TweenService:Create(alertFrame, TweenInfo.new(0.4, Enum.EasingStyle.Back, Enum.EasingDirection.In), {
                Position = UDim2.new(0.5, 0, 0, -130)
            }):Play()
            TweenService:Create(scaleObj, TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
                Scale = 0.85
            }):Play()
            task.wait(0.45)
            if alertGui and alertGui.Parent then alertGui:Destroy() end
        end
    end)
end

highestAlertedPriorityIndex = 99999

task.spawn(function()
    while true do
        task.wait(0.75)
        pcall(function()
            local cache = SharedState.AllAnimalsCache
            if not cache or #cache == 0 then return end
            
            local bestPet = nil
            local bestPriorityIndex = 99999
            
            -- O(cache + list) lookup map instead of the old nested loop
            -- (cache x priorityList string compares every 1.5s)
            local prioMap = {}
            for idx, pName in ipairs(priorityList) do
                local key = pName:lower()
                if not prioMap[key] then prioMap[key] = idx end
            end
            for _, pet in ipairs(cache) do
                local priorityIndex = prioMap[pet.name:lower()]
                if priorityIndex and priorityIndex < bestPriorityIndex then
                    bestPriorityIndex = priorityIndex
                    bestPet = pet
                end
            end
            
            if bestPet and bestPriorityIndex < highestAlertedPriorityIndex then
                local myName = LocalPlayer.Name
                local myDisplay = LocalPlayer.DisplayName
                if bestPet.owner ~= myName and bestPet.owner ~= myDisplay then
                    -- No 3D alert viewer mid-TP: cloning pet models into a
                    -- ViewportFrame during flight causes visible FPS spikes.
                    -- IMPORTANT: only mark as alerted when actually SHOWN --
                    -- marking while suppressed by a TP is why the startup
                    -- notification sometimes never appeared (auto-TP is often
                    -- already flying when the first scan lands). When
                    -- suppressed we simply retry on the next 1.5s pass.
                    if not _G._isTpMoving then
                        highestAlertedPriorityIndex = bestPriorityIndex
                        ShowPriorityAlertImpl(bestPet.name, bestPet.genText, bestPet.mutation, bestPet.owner, "PRIORITY BRAINROT")
                    end
                end
            end
        end)
    end
end)

-- SERVER-BEST WATCHER: shows the SERVER BEST BRAINROT notification the moment
-- the first scan data arrives, then updates it live whenever a new best appears.
task.spawn(function()
    local deadline = os.clock() + 15
    while os.clock() < deadline do
        local c = SharedState.AllAnimalsCache
        if c and #c > 0 then break end
        task.wait(0.5)
    end
    local function getServerBest()
        local myName, myDisplay = LocalPlayer.Name, LocalPlayer.DisplayName
        local best = nil
        local cache = SharedState.AllAnimalsCache
        if cache then
            for _, pet in ipairs(cache) do
                if pet.owner ~= myName and pet.owner ~= myDisplay then
                    if not best or (pet.genValue or 0) > (best.genValue or 0) then best = pet end
                end
            end
        end
        pcall(function()
            local conveyorFolder = Workspace:FindFirstChild("RenderedMovingAnimals")
            if not conveyorFolder then return end
            local Datas = ReplicatedStorage:FindFirstChild("Datas")
            local Shared = ReplicatedStorage:FindFirstChild("Shared")
            local Utils = ReplicatedStorage:FindFirstChild("Utils")
            if not (Datas and Shared and Utils) then return end
            local AnimalsData = require(Datas:WaitForChild("Animals", 5))
            local AnimalsShared = _G._xenAnimShim
            local NumberUtils = require(Utils:WaitForChild("NumberUtils", 5))
            for _, model in ipairs(conveyorFolder:GetChildren()) do
                pcall(function()
                    if not model:IsA("Model") then return end
                    local animalInfo = AnimalsData and AnimalsData[model.Name]
                    if not animalInfo then return end
                    local mutation = model:GetAttribute("Mutation") or "None"
                    if mutation == "Yin Yang" then mutation = "YinYang" end
                    local genValue = AnimalsShared:GetGeneration(model.Name, model:GetAttribute("Mutation"), nil, nil) or 0
                    if genValue <= 0 then return end
                    if not best or genValue > (best.genValue or 0) then
                        best = {
                            name = animalInfo.DisplayName or model.Name,
                            genText = "$" .. NumberUtils:ToString(genValue) .. "/s",
                            genValue = genValue,
                            mutation = mutation,
                            owner = "Conveyor",
                        }
                    end
                end)
            end
        end)
        return best
    end
    local lastBestVal = -1
    while true do
        local best = getServerBest()
        if best and (best.genValue or 0) > lastBestVal then
            lastBestVal = best.genValue
            ShowPriorityAlertImpl(best.name, best.genText, best.mutation, best.owner, "SERVER BEST BRAINROT")
        end
        task.wait(0.75)
    end
end)

-- ============================================================
-- PRIORITY SCANNER
-- ============================================================
PRIORITY_LIST = priorityList
task.spawn(function()
    local ok1,Packages=pcall(function() return ReplicatedStorage:WaitForChild("Packages",5) end); if not ok1 or not Packages then return end
    local ok2,Datas=pcall(function() return ReplicatedStorage:WaitForChild("Datas",5) end); if not ok2 or not Datas then return end
    local ok3,Shared=pcall(function() return ReplicatedStorage:WaitForChild("Shared",5) end); if not ok3 or not Shared then return end
    local ok4,Utils=pcall(function() return ReplicatedStorage:WaitForChild("Utils",5) end); if not ok4 or not Utils then return end
    local okS,Synchronizer=pcall(function() return require(Packages:WaitForChild("Synchronizer")) end); if not okS then return end
    local okA,AnimalsData=pcall(function() return require(Datas:WaitForChild("Animals")) end); if not okA then return end
    local AnimalsShared=_G._xenAnimShim
    local okN,NumberUtils=pcall(function() return require(Utils:WaitForChild("NumberUtils")) end); if not okN then return end
    local allAnimalsCache={}; local lastAnimalData={}
    local function normMut(m)if m=="None"or m==""or m==nil then return"NORMAL"end return m end
    local function getAnimalHash(al) if not al then return "" end; local h=""
        if #al>0 then for i=1,#al do local d=al[i]; if type(d)=="table" then h=h..tostring(i)..tostring(d.Index)..tostring(d.UUID or "")..tostring(d.Mutation or "")..tostring(d.Traits and table.concat(d.Traits,",")or"")end end
        else for slot,d in pairs(al)do if type(d)=="table"then h=h..tostring(slot)..tostring(d.Index)..tostring(d.Mutation or "")..tostring(d.Traits and table.concat(d.Traits,",")or"")end end end
    return h end
    local function scanSinglePlot(plot) pcall(function()
        local ch=_G.XenSyncGet(plot.Name); if not ch then return end
        local al=ch:Get("AnimalList"); local owner=ch:Get("Owner")
        if not owner or not owner.Name or not Players:FindFirstChild(owner.Name) then
            lastAnimalData[plot.Name]=nil; for i=#allAnimalsCache,1,-1 do if allAnimalsCache[i].plot==plot.Name then table.remove(allAnimalsCache,i) end end; return end
        if not al then lastAnimalData[plot.Name]=nil; for i=#allAnimalsCache,1,-1 do if allAnimalsCache[i].plot==plot.Name then table.remove(allAnimalsCache,i) end end; return end
        local ownerName=owner.Name; local hash=getAnimalHash(al)
        if lastAnimalData[plot.Name]==hash then return end
        
        for i=#allAnimalsCache,1,-1 do if allAnimalsCache[i].plot==plot.Name then table.remove(allAnimalsCache,i) end end
        if #al>0 then
            for slot=1,#al do local ad=al[slot]; if type(ad)=="table" and ad.Index~="Empty" and ad.Index~=nil then
                local aName,aInfo=ad.Index,AnimalsData[ad.Index]
                local displayName=aInfo and aInfo.DisplayName or aName
                local mut=ad.Mutation or "None"; if mut=="Yin Yang" then mut="YinYang" end
                local traits=ad.Traits
                local gv=aInfo and AnimalsShared:GetGeneration(aName,mut~="None" and mut or nil,traits) or 0
                if gv==0 then gv=aInfo and AnimalsShared:GetGeneration(aName,"None",nil) or 0 end
                local gt="$"..NumberUtils:ToString(gv).."/s"
                local pv=0
                table.insert(allAnimalsCache,{name=displayName,index=aName,genText=gt,genValue=gv,petValue=pv,mutation=normMut(mut),traits=traits and #traits>0 and table.concat(traits,", ")or"None",owner=ownerName,plot=plot.Name,slot=tostring(slot),uid=plot.Name.."_"..tostring(slot)})
            end end
        else
            for slot,ad in pairs(al) do if type(ad)=="table" then
                local aName,aInfo=ad.Index,AnimalsData[ad.Index]; if aInfo then
                    local displayName=aInfo.DisplayName or aName
                    local mut=ad.Mutation or "None"; if mut=="Yin Yang" then mut="YinYang" end
                    local traits=ad.Traits
                    local gv=AnimalsShared:GetGeneration(aName,mut~="None" and mut or nil,traits)
                    if not gv or gv==0 then gv=AnimalsShared:GetGeneration(aName,"None",nil) or 0 end
                    local gt="$"..NumberUtils:ToString(gv).."/s"
                    local pv=0
                    table.insert(allAnimalsCache,{name=displayName,index=aName,genText=gt,genValue=gv,petValue=pv,mutation=normMut(mut),traits=traits and #traits>0 and table.concat(traits,", ")or"None",owner=ownerName,plot=plot.Name,slot=tostring(slot),uid=plot.Name.."_"..tostring(slot)})
                end
            end end
        end
        lastAnimalData[plot.Name]=hash
        table.sort(allAnimalsCache,function(a,b) return a.genValue>b.genValue end)
        SharedState.AllAnimalsCache=allAnimalsCache; SharedState.ListNeedsRedraw=true
    end) end
    -- Brainrot scanner (brxken model): NO DescendantAdded/DescendantRemoving
    -- listeners. Those fired thousands of times while every plot's parts streamed
    -- in at startup -> that was the freeze/lag. Each plot instead gets ONE light
    -- 0.5s periodic rescan; scanSinglePlot is hash-gated, so it's a cheap no-op
    -- whenever a plot's AnimalList hasn't changed. retries softened (40 @ 0.07s)
    -- so the channel wait doesn't spin tight during load.
    local function setupPlotListener(plot) local ch; local retries=0
        while not ch and retries<40 do local ok,r=pcall(function() return _G.XenSyncGet(plot.Name) end); if ok and r then ch=r; break else retries=retries+1; task.wait(0.07) end end
        if not ch then return end; scanSinglePlot(plot)
        task.spawn(function() while plot.Parent do task.wait(0.25); scanSinglePlot(plot) end end)
    end
    local plots=Workspace:WaitForChild("Plots",8)
    if plots then
        for _,p in ipairs(plots:GetChildren()) do
            task.spawn(setupPlotListener, p)
        end
        SharedState.InitialScanComplete=true
        plots.ChildAdded:Connect(function(p) task.spawn(setupPlotListener, p) end)
        plots.ChildRemoved:Connect(function(p) lastAnimalData[p.Name]=nil; for i=#allAnimalsCache,1,-1 do if allAnimalsCache[i].plot==p.Name then table.remove(allAnimalsCache,i) end end; SharedState.ListNeedsRedraw=true end)
    end
    task.spawn(function() while true do SharedState.AllAnimalsCache=allAnimalsCache; task.wait(0.25) end end)
end)

-- Steal attribute listeners
player:GetAttributeChangedSignal("Stealing"):Connect(function()
    local isStealing=(player:GetAttribute("Stealing")==true)
    -- Float auto-off the moment a steal STARTS (user request: never float mid-steal)
    if FloatState.active and isStealing then setFloat(false) end
    if _G.AutoInvisDuringSteal then
        if isStealing and not _G.invisibleStealEnabled and _G._forceInvisToggle then task.defer(function() if player:GetAttribute("Stealing") and not _G.invisibleStealEnabled then pcall(_G._forceInvisToggle) end end)
        elseif not isStealing and _G.invisibleStealEnabled and _G._forceInvisToggle then task.wait(0.3); if not player:GetAttribute("Stealing") then pcall(_G._forceInvisToggle) end end
    end
    if isStealing and Config.AutoUnlockOnSteal then
        local hrp=player.Character and player.Character:FindFirstChild("HumanoidRootPart")
        if hrp then
            local floor = getCurrentUnlockFloor()
            task.spawn(function()
                task.wait(0.1)
                pcall(smartInteract, floor)
            end)
        end
    end
end)

player.CharacterAdded:Connect(function() task.wait(0.5); if FloatState.active then removeFloatPlatform(); createFloatPlatform() end end)

-- ============================================================
-- TP SYSTEM
-- ============================================================
function findAdorneeGlobal(animalData)
    if not animalData then return nil end
    local plot = Workspace:FindFirstChild("Plots") and Workspace.Plots:FindFirstChild(animalData.plot)
    if plot then
        local podiums = plot:FindFirstChild("AnimalPodiums")
        if podiums then
            local podium = podiums:FindFirstChild(animalData.slot)
            if podium then
                local base = podium:FindFirstChild("Base")
                if base then
                    local spawn = base:FindFirstChild("Spawn")
                    if spawn then return spawn end
                    return base:FindFirstChildWhichIsA("BasePart") or base
                end
            end
        end
    end
    return nil
end

function getClosestBaseSign(brainrotPart)
    if not brainrotPart or not brainrotPart:IsA("BasePart") then return nil end
    local closestPart = nil
    local closestDist = math.huge
    for _, label in ipairs(Workspace:GetDescendants()) do
        if label:IsA("TextLabel") then
            local txt = tostring(label.Text or "")
            if (txt ~= "") and txt:lower():find("base", 1, true) then
                local gui = label:FindFirstAncestorWhichIsA("SurfaceGui")
                if gui then
                    local part = gui.Adornee or gui.Parent
                    if part and part:IsA("BasePart") then
                        local dist = (part.Position - brainrotPart.Position).Magnitude
                        if dist < closestDist then
                            closestDist = dist
                            closestPart = part
                        end
                    end
                end
            end
        end
    end
    return closestPart
end

function riseToY(hrp, targetY)
    if not hrp then return end
    local MAX_TIME = 3
    local start = os.clock()
    while hrp.Parent and (hrp.Position.Y < targetY) do
        local dist = targetY - hrp.Position.Y
        local speed = math.clamp(dist * 20, 280, 310)
        hrp.AssemblyLinearVelocity = Vector3.new(hrp.AssemblyLinearVelocity.X, speed, hrp.AssemblyLinearVelocity.Z)
        if (os.clock() - start) > MAX_TIME then
            break
        end
        RunService.Heartbeat:Wait()
    end
    hrp.AssemblyLinearVelocity = Vector3.new(hrp.AssemblyLinearVelocity.X, 0, hrp.AssemblyLinearVelocity.Z)
end

function equipTpToolAndWait(hum)
    if not hum then return nil end
    local char = hum.Parent
    local toolName = Config.TpSettings.Tool or "Flying Carpet"
    local tool = LocalPlayer.Backpack:FindFirstChild(toolName) or (char and char:FindFirstChild(toolName))
    if tool then
        hum:EquipTool(tool)
        task.wait(0.02)
    end
    return tool
end

function walkForward(seconds)
    local char = LocalPlayer.Character
    local hum = char:FindFirstChild("Humanoid")
    local hrp = char:FindFirstChild("HumanoidRootPart")
    local pm = LocalPlayer:WaitForChild("PlayerScripts")
    local Controls = require(pm:WaitForChild("PlayerModule")):GetControls()
    local lookVector = hrp.CFrame.LookVector
    Controls:Disable()
    local startTime = os.clock()
    local conn
    conn = RunService.RenderStepped:Connect(function()
        if (os.clock() - startTime) >= seconds then
            conn:Disconnect()
            hum:Move(Vector3.zero, false)
            Controls:Enable()
            return
        end
        hum:Move(lookVector, false)
    end)
end

function waitSecondsHeartbeat(sec)
    local t = 0
    while t < sec do
        t += RunService.Heartbeat:Wait()
    end
end

function waitUntilHeartbeat(predicate, timeoutSec)
    local t = 0
    while true do
        if predicate() then
            return true
        end
        local dt = RunService.Heartbeat:Wait()
        t += dt
        if timeoutSec and (t >= timeoutSec) then
            return false
        end
    end
end

-- ============================================================
-- TELEPORT SYSTEM STAGE 2
-- ============================================================
TP_V2_MED_POINTS = {
    { name = "MED1", pos = Vector3.new(-410.65, -5.68, -46.1) },
    { name = "MED2", pos = Vector3.new(-410.91, -5.68, 168.89) },
}

TP_V2_SECOND_FLOOR_POINTS = {
    { name = "TP1", pos = Vector3.new(-488.88, 15, 196.38), facing = "back" },
    { name = "TP2", pos = Vector3.new(-487.79, 15, 138.13), facing = "front" },
    { name = "TP3", pos = Vector3.new(-489.38, 15, 89.23), facing = "back" },
    { name = "TP4", pos = Vector3.new(-489.69, 15, 30.98), facing = "front" },
    { name = "TP5", pos = Vector3.new(-488.75, 15, -17.95), facing = "back" },
    { name = "TP6", pos = Vector3.new(-490, 15, -75.9), facing = "front" },
    { name = "TP7", pos = Vector3.new(-331.75, 15, -75.8), facing = "back" },
    { name = "TP8", pos = Vector3.new(-329.98, 15, -18.16), facing = "front" },
    { name = "TP9", pos = Vector3.new(-330.04, 15, 31.14), facing = "back" },
    { name = "TP10", pos = Vector3.new(-331.28, 15, 88.92), facing = "front" },
    { name = "TP11", pos = Vector3.new(-330.57, 15, 138.1), facing = "back" },
    { name = "TP12", pos = Vector3.new(-330.01, 15, 195.96), facing = "front" },
}

TP_V2_ALLOWED_BY_MED = {
    MED1 = { TP6 = true, TP7 = true, TP8 = true, TP10 = true, TP12 = true, TP5 = true, TP3 = true, TP1 = true },
    MED2 = { TP1 = true, TP2 = true, TP4 = true, TP6 = true, TP7 = true, TP9 = true, TP11 = true, TP12 = true },
}

function flatDistance(a, b)
    return (Vector3.new(a.X, 0, a.Z) - Vector3.new(b.X, 0, b.Z)).Magnitude
end

_G._isTargetPlotUnlocked = function(plotName)
    local ok, res = pcall(function()
        local plots = Workspace:FindFirstChild("Plots")
        if not plots then return false end
        local targetPlot = plots:FindFirstChild(plotName)
        if not targetPlot then return false end
        local unlockFolder = targetPlot:FindFirstChild("Unlock")
        if not unlockFolder then return true end
        local unlockItems = {}
        for _, item in pairs(unlockFolder:GetChildren()) do
            local pos = nil
            if item:IsA("Model") then
                pcall(function() pos = item:GetPivot().Position end)
            elseif item:IsA("BasePart") then
                pos = item.Position
            end
            if pos then
                table.insert(unlockItems, { Object = item, Height = pos.Y })
            end
        end
        table.sort(unlockItems, function(a, b) return a.Height < b.Height end)
        if #unlockItems == 0 then return true end
        local floor1Door = unlockItems[1].Object
        for _, desc in ipairs(floor1Door:GetDescendants()) do
            if desc:IsA("ProximityPrompt") and desc.Enabled then return false end
        end
        for _, child in ipairs(floor1Door:GetChildren()) do
            if child:IsA("ProximityPrompt") and child.Enabled then return false end
        end
        return true
    end)
    return (ok and res) or false
end

function getClosestBaseSignToPosition(worldPos)
    local closestPart = nil
    local closestDist = math.huge
    for _, label in ipairs(Workspace:GetDescendants()) do
        if label:IsA("TextLabel") then
            local txt = tostring(label.Text or "")
            if (txt ~= "") and txt:lower():find("base", 1, true) then
                local gui = label:FindFirstAncestorWhichIsA("SurfaceGui")
                if gui then
                    local part = gui.Adornee or gui.Parent
                    if part and part:IsA("BasePart") then
                        local dist = (part.Position - worldPos).Magnitude
                        if dist < closestDist then
                            closestDist = dist
                            closestPart = part
                        end
                    end
                end
            end
        end
    end
    return closestPart
end

function getNearestTeleportV2MedPoint(fromPos)
    local bestPoint = nil
    local bestDist = math.huge
    for _, entry in ipairs(TP_V2_MED_POINTS) do
        local dist = flatDistance(fromPos, entry.pos)
        if dist < bestDist then
            bestDist = dist
            bestPoint = entry
        end
    end
    return bestPoint, bestDist
end

function getBestTeleportV2SecondFloorPoint(medPoint, brainrotPos)
    local medPos = medPoint and medPoint.pos
    local medName = medPoint and medPoint.name
    if not medPos or not medName then return nil end
    local allowed = TP_V2_ALLOWED_BY_MED[medName]
    if not allowed then return nil end
    local bestPoint = nil
    local bestMedDist = math.huge
    local bestBrainrotDist = math.huge
    for _, entry in ipairs(TP_V2_SECOND_FLOOR_POINTS) do
        if allowed[entry.name] then
            local distToMed = flatDistance(medPos, entry.pos)
            local distToBrainrot = flatDistance(brainrotPos, entry.pos)
            if (distToBrainrot < bestBrainrotDist) or ((math.abs(distToBrainrot - bestBrainrotDist) <= 0.001) and (distToMed < bestMedDist)) then
                bestPoint = entry
                bestMedDist = distToMed
                bestBrainrotDist = distToBrainrot
            end
        end
    end
    return bestPoint, bestMedDist, bestBrainrotDist
end



FLY_SPEED = 160
FLY_RISE_SPEED = 200
FLY_RAY_DIST = 20
FLY_TIMEOUT = 15
FLY_ARRIVE_DIST = 4
flyRayParams = RaycastParams.new()
flyRayParams.FilterType = Enum.RaycastFilterType.Exclude

function flyForwardTo(hrp, tpPos, lookDir, targetY, customSpeed)
    if not hrp or not hrp.Parent then
        return false
    end
    local char = hrp.Parent
    local hum = char:FindFirstChildOfClass("Humanoid")
    local unwalkConn
    if hum then
        local animator = hum:FindFirstChildOfClass("Animator")
        if animator then
            unwalkConn = RunService.Heartbeat:Connect(function()
                pcall(function()
                    for _, tr in ipairs(animator:GetPlayingAnimationTracks()) do
                        tr:Stop()
                    end
                end)
            end)
        end
    end
    flyRayParams.FilterDescendantsInstances = { char }
    local startTime = os.clock()
    while hrp.Parent do
        if (os.clock() - startTime) > FLY_TIMEOUT then
            break
        end
        if LocalPlayer:GetAttribute("Stealing") then
            break
        end
        local myPos = hrp.Position
        local flatDiff = Vector3.new(tpPos.X - myPos.X, 0, tpPos.Z - myPos.Z)
        local flatDist = flatDiff.Magnitude
        if flatDist < FLY_ARRIVE_DIST then
            break
        end
        local moveDir = flatDiff.Unit
        local yVel = 0
        if tpPos.Y <= 10 then
            -- Glue the feet to the floor by offsetting the HumanoidRootPart by +3.5 studs to prevent clipping into the ground and dying
            local targetFloorY = tpPos.Y + 3.5
            if targetY then
                -- Diagonal smooth ascent starting from 60 studs away down to 20 studs away
                if flatDist <= 60 then
                    local t = math.clamp((60 - flatDist) / (60 - 20), 0, 1)
                    targetFloorY = (tpPos.Y + 3.5) + (targetY - tpPos.Y) * t
                end
            end
            -- Multi-Ray Collision Detection (Feet, Torso, and Head levels)
            local heightsToCheck = {-3, 0, 3}
            local isRealObstacle = false
            for _, heightOffset in ipairs(heightsToCheck) do
                local rayOrigin = myPos + Vector3.new(0, heightOffset, 0)
                local rayFwd = Workspace:Raycast(rayOrigin, moveDir * FLY_RAY_DIST, flyRayParams)
                if rayFwd then
                    local part = rayFwd.Instance
                    if part and part.CanCollide then
                        local topY = part.Position.Y + (part.Size.Y / 2)
                        -- Only fly over if the obstacle's top is significantly higher than flat ground (more than 5 studs)
                        if (topY - tpPos.Y) > 5 then
                            isRealObstacle = true
                            break -- Tall obstacle detected, stop checking other heights
                        end
                    end
                end
            end

            if isRealObstacle then
                -- If tall obstacle detected, rise up and fly over it
                yVel = FLY_RISE_SPEED
            else
                -- Check if there is a real tall obstacle underneath us
                local rayDown = Workspace:Raycast(myPos, Vector3.new(0, -15, 0), flyRayParams)
                local obstacleUnderneath = false
                if rayDown then
                    local hitY = rayDown.Position.Y
                    if hitY > (tpPos.Y + 4) then
                        obstacleUnderneath = true
                    end
                end

                if obstacleUnderneath then
                    -- Keep current altitude while flying over the obstacle
                    yVel = 0
                else
                    -- No real obstacle in front or underneath, instantly stick to the ground
                    hrp.CFrame = CFrame.new(hrp.Position.X, targetFloorY, hrp.Position.Z) * (hrp.CFrame - hrp.CFrame.Position)
                    yVel = 0
                end
            end
        else
            -- Normal rise/fall behavior for second floor
            local rayFwd = Workspace:Raycast(myPos, moveDir * FLY_RAY_DIST, flyRayParams)
            if rayFwd then
                yVel = FLY_RISE_SPEED
            end
            local rayDown = Workspace:Raycast(myPos, Vector3.new(0, -12, 0), flyRayParams)
            if rayDown and ((myPos.Y - rayDown.Position.Y) < 5) then
                yVel = math.max(yVel, 100)
            end
            if not rayFwd and (tpPos.Y < (myPos.Y - 3)) then
                yVel = math.clamp((tpPos.Y - myPos.Y) * 3, -120, 0)
            end
        end
        local speed = customSpeed or Config.TpSettings.FlyTPSpeed or FLY_SPEED
        hrp.AssemblyLinearVelocity = (moveDir * speed) + Vector3.new(0, yVel, 0)
        hrp.AssemblyAngularVelocity = Vector3.zero
        RunService.Heartbeat:Wait()
    end
    if unwalkConn then
        unwalkConn:Disconnect()
        unwalkConn = nil
    end
    if not hrp.Parent then
        return false
    end
    hrp.AssemblyLinearVelocity = Vector3.zero
    hrp.AssemblyAngularVelocity = Vector3.zero
    local finalY = targetY or tpPos.Y
    local finalPos = Vector3.new(tpPos.X, finalY, tpPos.Z)
    hrp.CFrame = CFrame.lookAt(finalPos, finalPos + (lookDir or hrp.CFrame.LookVector))
    hrp.AssemblyLinearVelocity = Vector3.zero
    return true
end

function prepMiniTpTool(hum, hrp)
    if not hum or not hrp then return end
end

function getTargetPetData()
    local cache = SharedState.AllAnimalsCache
    if not cache or #cache == 0 then
        return nil
    end
    
    local pets = get_all_pets()
    
    -- 1) Manual Override First
    if manuallySelectedUID then
        for _, a in ipairs(cache) do
            if a.uid == manuallySelectedUID and a.owner ~= LocalPlayer.Name then
                return a
            end
        end
    end

    -- 1.5) Priority List FIRST (Always target priority pets before anything else)
    for _, pName in ipairs(priorityList) do
        local searchName = pName:lower()
        local bestPet = nil
        local bestDist = math.huge
        for _, a in ipairs(cache) do
            if a and a.name and ((a.name:lower() == searchName) or (a.index and a.index:lower() == searchName)) and (a.owner ~= LocalPlayer.Name) then
                local dist = math.huge
                local char = LocalPlayer.Character
                local hrp = char and char:FindFirstChild("HumanoidRootPart")
                if hrp then
                    local plots = Workspace:FindFirstChild("Plots")
                    local plotInst = plots and plots:FindFirstChild(a.plot)
                    if plotInst then
                        local pos = nil
                        pcall(function() pos = plotInst:GetPivot().Position end)
                        if pos then
                            dist = (hrp.Position - pos).Magnitude
                        end
                    end
                end
                
                if not bestPet then
                    bestPet = a
                    bestDist = dist
                else
                    local currentGen = a.genValue or 0
                    local bestGen = bestPet.genValue or 0
                    if currentGen > bestGen then
                        bestPet = a
                        bestDist = dist
                    elseif currentGen == bestGen then
                        if dist < bestDist then
                            bestPet = a
                            bestDist = dist
                        end
                    end
                end
            end
        end
        if bestPet then
            return bestPet
        end
    end

    -- 2) Brainrots Second (target brainrots after priority)
    -- Respect Min-Gen-for-TP here too. Without this, brainrots BELOW the configured
    -- min gen (but still over the 10M "is-a-brainrot" baseline) got teleported to
    -- even though the list/scan correctly excluded them. Priority pets above are
    -- intentionally exempt from min gen; this non-priority section is not.
    local _mgStr = (Config.StealNearest and Config.TpSettings.MinGenForGrab) or Config.TpSettings.MinGenForTp
    local _brainrotFloor = math.max(10000000, parseMinGen(_mgStr) or 0)
    local bestBrainrot = nil
    local bestBrainrotDist = math.huge
    for _, a in ipairs(cache) do
        if a and a.owner ~= LocalPlayer.Name and a.genValue and a.genValue >= _brainrotFloor then
            local dist = math.huge
            local char = LocalPlayer.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if hrp then
                local plots = Workspace:FindFirstChild("Plots")
                local plotInst = plots and plots:FindFirstChild(a.plot)
                if plotInst then
                    local pos = nil
                    pcall(function() pos = plotInst:GetPivot().Position end)
                    if pos then dist = (hrp.Position - pos).Magnitude end
                end
            end
            if dist < bestBrainrotDist then
                bestBrainrotDist = dist
                bestBrainrot = a
            end
        end
    end
    if bestBrainrot then return bestBrainrot end

    -- 3) Auto TP Highest Gen
    if Config.AutoTPHighestGen then
        if pets and #pets > 0 then
            return pets[1].animalData
        end
    end
    
    -- 2b) Auto TP Highest Value (sorted by pet value, not gen)
    if Config.AutoTPHighestValue then
        local valuePets = get_all_pets_by_value()
        if valuePets and #valuePets > 0 then
            return valuePets[1].animalData
        end
    end
    
    -- 3b) Auto TP Priority mode fallback (only reached if no priority pet was found)
    -- If they specifically selected priority mode but no priority pet is on the server, we return the highest gen fallback
    if Config.AutoTPPriority then
        if pets and #pets > 0 then
            return pets[1].animalData
        end
        return nil
    end
    
    -- 4) General Selected Pet
    if SharedState.SelectedPetData then
        return SharedState.SelectedPetData.animalData
    end
    
    return nil
end

do
UPPER = {
    B = {{coord=Vector3.new(-487.921448,16.850713,-75.768013),facing="NORTH"},{coord=Vector3.new(-332.379730,16.850722,-75.762100),facing="NORTH"},{coord=Vector3.new(-487.134918,16.850713,-18.094154),facing="SOUTH"},{coord=Vector3.new(-316.300171,16.850713,-17.845898),facing="SOUTH"}},
    C = {{coord=Vector3.new(-330.765381,16.850713,31.424425),facing="NORTH"},{coord=Vector3.new(-502.989349,16.850713,31.172430),facing="NORTH"},{coord=Vector3.new(-489.077087,16.850713,89.010147),facing="SOUTH"},{coord=Vector3.new(-330.908936,16.850713,88.930145),facing="SOUTH"}},
    D = {{coord=Vector3.new(-331.264893,16.850713,138.209167),facing="NORTH"},{coord=Vector3.new(-487.935181,16.850713,138.026321),facing="NORTH"},{coord=Vector3.new(-487.774933,16.850713,195.882538),facing="SOUTH"},{coord=Vector3.new(-330.799133,16.850575,196.022354),facing="SOUTH"}},
}
LOWER = {
    B = {{coord=Vector3.new(-335.725586,-3.048217,-74.984589),facing="NORTH"},{coord=Vector3.new(-503.214233,-3.048217,-75.043137),facing="NORTH"},{coord=Vector3.new(-483.619385,-3.718430,-18.844337),facing="SOUTH"},{coord=Vector3.new(-316.147095,-3.048218,-18.818844),facing="SOUTH"}},
    C = {{coord=Vector3.new(-335.985413,-3.048218,32.051426),facing="NORTH"},{coord=Vector3.new(-503.277008,-3.048217,31.956175),facing="NORTH"},{coord=Vector3.new(-483.749390,-3.048218,88.147003),facing="SOUTH"},{coord=Vector3.new(-315.793823,-3.048217,88.163979),facing="SOUTH"}},
    D = {{coord=Vector3.new(-335.476654,-3.048218,139.001083),facing="NORTH"},{coord=Vector3.new(-503.710083,-3.048218,138.989883),facing="NORTH"},{coord=Vector3.new(-315.654938,-3.048218,195.302444),facing="SOUTH"},{coord=Vector3.new(-483.859253,-3.048218,195.269043),facing="SOUTH"}},
}
UPPER_Y_THRESHOLD = 7
TALL_PETS = { ["La Secret Combinasion"]=true, ["La Jolly Grande"]=true }
TALL_OFFSET = 3

CARPET_SPEED = 230
INBASE_SPEED = 230
function getCarpetSpeed()
    return Config.TpSettings.FlyTPSpeed or 230
end
function getInBaseSpeed()
    return Config.TpSettings.FlyTPCloseSpeed or 230
end
SKY_CLONE_WAIT = 0.2
CARPET_NAMES = { "Flying Carpet", "Carpet", "Cloud", "Witch's Broom", "Cupid's Wings", "Santa's Sleigh", "Magic Carpet", "Waverider" }
GRAPPLE_NAMES = { "Grapple Hook", "Grappling Hook", "Grapple", "Hook", "Web Slinger", "Grapple Gun", "GrappleHook" }

local Players    = game:GetService("Players")
local RunService = game:GetService("RunService")
UIS        = game:GetService("UserInputService")
RS         = game:GetService("ReplicatedStorage")
LP         = Players.LocalPlayer

_G.AntiDieDisabled = false
do
    local _conn, _diedConn, _hbConn
    local _harden = function(hum)
        pcall(function() hum.BreakJointsOnDeath = false end)
        pcall(function() hum.RequiresNeck = false end)
        pcall(function() hum:SetStateEnabled(Enum.HumanoidStateType.Dead, false) end)
        pcall(function() hum:SetStateEnabled(Enum.HumanoidStateType.Ragdoll, false) end)
        pcall(function() hum:SetStateEnabled(Enum.HumanoidStateType.FallingDown, false) end)
        pcall(function() hum:SetStateEnabled(Enum.HumanoidStateType.Physics, false) end)
    end
    local function _revive(hum)
        pcall(function() hum.Health = hum.MaxHealth end)
        pcall(function() hum:ChangeState(Enum.HumanoidStateType.Running) end)
    end
    local function _bind()
        local char = LP.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not hum then return end
        _harden(hum)
        if _conn then pcall(function() _conn:Disconnect() end) end
        if _diedConn then pcall(function() _diedConn:Disconnect() end) end
        if _hbConn then pcall(function() _hbConn:Disconnect() end) end
        _conn = hum:GetPropertyChangedSignal("Health"):Connect(function()
            if _G.AntiDieDisabled then return end
            if hum.Health <= 0 then _revive(hum) end
        end)
        _diedConn = hum.Died:Connect(function()
            if _G.AntiDieDisabled then return end
            _revive(hum)
        end)
        local _lastHarden = 0
        _hbConn = RunService.Heartbeat:Connect(function()
            if _G.AntiDieDisabled or not hum or not hum.Parent then return end
            local now = os.clock()
            if now - _lastHarden >= 0.5 then _lastHarden = now; _harden(hum) end
            if hum.Health <= 0 then _revive(hum) end
            local state = hum:GetState()
            if state == Enum.HumanoidStateType.Dead or state == Enum.HumanoidStateType.Ragdoll
               or state == Enum.HumanoidStateType.FallingDown then
                pcall(function() hum:ChangeState(Enum.HumanoidStateType.Running) end)
            end
        end)
    end
    _bind()
    LP.CharacterAdded:Connect(function(char)
        local hum = char:WaitForChild("Humanoid", 5)
        if hum then _harden(hum) end
        task.wait(0.1)
        _bind()
    end)
end



-- =====================================================================
-- Module loaders
-- =====================================================================
local Synchronizer, AnimalsData, AnimalsShared, NumberUtils

function loadModules()
    if Synchronizer then return true end
    local ok = pcall(function()
        local Packages = RS:WaitForChild("Packages", 5)
        local Datas = RS:WaitForChild("Datas", 5)
        local Shared = RS:WaitForChild("Shared", 5)
        local Utils = RS:WaitForChild("Utils", 5)
        Synchronizer = require(Packages:WaitForChild("Synchronizer"))
        AnimalsData = require(Datas:WaitForChild("Animals"))
        AnimalsShared = _G._xenAnimShim
        NumberUtils = require(Utils:WaitForChild("NumberUtils"))
    end)
    return ok and Synchronizer ~= nil
end

function loadNet()
    if NetModule then return true end
    local ok, mod = pcall(function()
        return _G.Net
    end)
    if not ok or type(mod) ~= "table" then return false end
    NetModule = mod
    return true
end

function findTool(name)
    local char = LP.Character
    local bp = LP:FindFirstChild("Backpack")
    return (char and char:FindFirstChild(name)) or (bp and bp:FindFirstChild(name))
end

function findGrapple()
    for _, n in ipairs(GRAPPLE_NAMES) do
        local t = findTool(n)
        if t and t:IsA("Tool") then return t, n end
    end
    return nil
end

function equipCarpet()
    local char = LP.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not hum then return nil end
    local preferred = Config and Config.TpSettings and Config.TpSettings.Tool
    if preferred then
        local pt = findTool(preferred)
        if pt and pt:IsA("Tool") then
            if pt.Parent ~= char then pcall(function() hum:EquipTool(pt) end) end
            return preferred
        end
    end
    for _, n in ipairs(CARPET_NAMES) do
        local t = findTool(n)
        if t and t:IsA("Tool") then
            if t.Parent ~= char then pcall(function() hum:EquipTool(t) end) end
            return n
        end
    end
    return nil
end

function fireGrapple()
    if not NetModule then loadNet() end
    if not NetModule then return end
    local char = LP.Character
    if not char then return end
    if not char:FindFirstChild("Grapple Hook") then
        local bp = LP:FindFirstChild("Backpack")
        local tool = bp and bp:FindFirstChild("Grapple Hook")
        local hum = char:FindFirstChildOfClass("Humanoid")
        if tool and hum then pcall(function() hum:EquipTool(tool) end) end
    end
    if not char:FindFirstChild("Grapple Hook") then return end
    pcall(function() NetModule:RemoteEvent("UseItem"):FireServer(2) end)
end

function carpetEngage()
    if not NetModule then pcall(loadNet) end
    local char = LP.Character
    local hum = char and char:FindFirstChildOfClass("Humanoid")
    if not char or not hum then return nil end
    local g = findTool("Grapple Hook")
    if g then
        if g.Parent ~= char then 
            pcall(function() hum:EquipTool(g) end)
            task.wait(0.05) -- Let tool equip register
        end
        if NetModule then pcall(function() NetModule:RemoteEvent("UseItem"):FireServer(2) end) end
        task.wait(0.22) -- Let grapple projectile register and pull
    end
    pcall(function() hum:UnequipTools() end)
    task.wait(0.10) -- Wait before putting carpet on
    local cn = equipCarpet()
    _G.TPEngage = "carpet=" .. tostring(cn)
    return cn
end

-- =====================================================================
-- Pet priority data tables
-- =====================================================================
PET_PRIORITY_TIERS = {
    [1] = { pets = {"Headless Horseman"}, threshold = 0 },
    [2] = { pets = {"Signore Carapace"}, threshold = 0 },
    [3] = { pets = {"John Pork"}, threshold = 0 },
    [4] = { pets = {"Strawberry Elephant"}, threshold = 0 },
    [5] = { pets = {"Arcadragon"}, threshold = 5e9 },
    [6] = { pets = {"Elefanto Frigo"}, threshold = 10e9 },
    [7] = { pets = {"Meowl"}, threshold = 5e9 },
    [8] = { pets = {"Skibidi Toilet"}, threshold = 5e9 },
    [9] = { pets = {"Love Love Bear"}, threshold = 0 },
    [10] = { pets = {"Antonio"}, threshold = 0 },
    [11] = { pets = {"Pancake and Syrup"}, threshold = 0 },
    [12] = { pets = {"Griffin"}, threshold = 0 },
    [13] = { pets = {"Globa Steppa","La Supreme Combinasion","Fishino Clownino","Dragon Gingerini","Tirilikalika Tirilikalako"}, threshold = 5e9 },
    [14] = { pets = {"Ginger Gerat","Pet"}, threshold = 10e9 },
    [15] = { pets = {"Hydra Bunny","Digi Narwhal","Kalika Bros"}, threshold = 3e9 },
    [16] = { pets = {"Hydra Dragon Cannelloni","Dragon Cannelloni","Bunny and Eggy"}, threshold = 3e9 },
    [17] = { pets = {"Ketupat Bros","Rosey and Teddy","La Casa Boo","Fragola la la"}, threshold = 3e9 },
    [18] = { pets = {"Fragola La La La","Cerberus","Guest 666","Los Hackers"}, threshold = 1e9 },
    [19] = { pets = {"Garama and Madundung","Spooky and Pumpky","Reinito Sleighito","Burguro And Fryuro","Cooki and Milki","Fragrama and Chocrama","La Food Combinasion","Los Amigos","Foxini Lanternini","Capitano Moby","Fortunu and Cashuru","Los Sekolahs","Celestial Pegasus"}, threshold = 750e6 },
    [20] = { pets = {"La Secret Combinasion","Sammyni Fattini","Cloverat Clapat","Popcuru and Fizzuru"}, threshold = 1e9 },
}

TIER_LOOKUP = {}
for tier, data in pairs(PET_PRIORITY_TIERS) do
    for _, name in ipairs(data.pets) do TIER_LOOKUP[name] = tier end
end

LOCKED_TIERS = { [1]=true, [2]=true, [3]=true, [4]=true }

DIRECT_THRESHOLDS = {
    [3] = { [4] = 10e9 },
    [4] = {},
    [5] = { [6] = math.huge },
    [6] = { [9] = math.huge, [10] = math.huge, [12] = 15e9 },
    [10] = { [12] = 20e9 },
    [11] = { [12] = 10e9 },
}

MUTATION_PRIORITY = {
    ["Galaxy"]=1,["Candy"]=1,["Yin Yang"]=1,["YinYang"]=1,["Divine"]=1,
    ["Cursed"]=1,["Lava"]=1,["Radioactive"]=1,["Cyber"]=1,["Rainbow"]=1,["Bloodrot"]=2,
}

MUTATED_BEATS_GRIFFIN = {
    ["Fishino Clownino"]=true,["Globa Steppa"]=true,
    ["La Supreme Combinasion"]=true,["Tirilikalika Tirilikalako"]=true,
}

function getMutPrio(m)
    if not m or m == "" or m == "None" then return 0 end
    if MUTATION_PRIORITY[m] then return MUTATION_PRIORITY[m] end
    local n = tostring(m):lower():gsub("[%s%-_]","")
    if n == "bloodrot" then return 2 end
    if n == "yinyang" or n == "galaxy" or n == "candy" or n == "divine"
        or n == "cursed" or n == "lava" or n == "radioactive" or n == "cyber"
        or n == "rainbow" then return 1 end
    return 0
end

function getCumThreshold(hi, lo)
    if DIRECT_THRESHOLDS[hi] and DIRECT_THRESHOLDS[hi][lo] then return DIRECT_THRESHOLDS[hi][lo] end
    if LOCKED_TIERS[hi] then return math.huge end
    local total = 0
    for t = hi + 1, lo do
        local td = PET_PRIORITY_TIERS[t]
        if td and td.threshold > 0 then total = total + td.threshold end
    end
    return total
end

function petOutranks(aName, bName, aMut, bMut, aMPS, bMPS)
    if aName == "Strawberry Elephant" and bName == "John Pork" then return true end
    if aName == "John Pork" and bName == "Strawberry Elephant" then return false end

    if MUTATED_BEATS_GRIFFIN[aName] and bName == "Griffin" and getMutPrio(aMut) >= 1 then return true end
    if aName == "Griffin" and MUTATED_BEATS_GRIFFIN[bName] and getMutPrio(bMut) >= 1 then return false end

    if aName == "Antonio" and bName == "Elefanto Frigo" and getMutPrio(aMut) >= 1 then return true end
    if aName == "Elefanto Frigo" and bName == "Antonio" and getMutPrio(bMut) >= 1 then return false end

    local tA = TIER_LOOKUP[aName] or 99
    local tB = TIER_LOOKUP[bName] or 99

    if not (TIER_LOOKUP[aName] and TIER_LOOKUP[bName]) then
        if tA == tB then return (aMPS or 0) > (bMPS or 0) end
        return tA < tB
    end

    if tA == tB then
        local pA, pB = getMutPrio(aMut), getMutPrio(bMut)
        if pA ~= pB then return pA > pB end
        return (aMPS or 0) > (bMPS or 0)
    end

    if tA == 4 and tB == 3 then return true end
    if tA == 3 and tB == 4 then return false end

    local hi = math.min(tA, tB)
    local lo = math.max(tA, tB)
    local hiMPS = tA < tB and aMPS or bMPS
    local loMPS = tA < tB and bMPS or aMPS
    local cum = getCumThreshold(hi, lo)
    if cum > 0 and cum ~= math.huge then
        if (loMPS or 0) - (hiMPS or 0) > cum then return tA > tB end
    end
    return tA < tB
end

-- =====================================================================
-- Plot / channel helpers
-- =====================================================================
function getPlotChannel(plotName)
    if not Synchronizer then return nil end
    local channel
    pcall(function() channel = _G.XenSyncGet(plotName) end)
    if not channel then local _t0=os.clock() repeat channel=_G.XenSyncGet(plotName) if channel then break end task.wait(0.05) until os.clock()-_t0>3 end
    return channel
end

function channelGet(channel, key)
    return _G.sProp(channel, key)
end

function isMyPlot(channel)
    if not channel then return false end
    local owner = channelGet(channel, "Owner")
    if not owner then return false end
    local result = false
    pcall(function()
        if typeof(owner) == "Instance" and owner:IsA("Player") then
            result = owner.UserId == LP.UserId
        elseif type(owner) == "table" and owner.UserId then
            result = owner.UserId == LP.UserId
        elseif typeof(owner) == "Instance" then
            result = owner == LP
        elseif type(owner) == "string" then
            result = (owner:lower() == LP.Name:lower() or owner:lower() == LP.DisplayName:lower())
        elseif type(owner) == "number" then
            result = owner == LP.UserId
        end
    end)
    return result
end

function ownerInGame(channel)
    if not channel then return false end
    local owner = channelGet(channel, "Owner")
    if not owner then return false end
    local inGame = false
    pcall(function()
        if typeof(owner) == "Instance" and owner:IsA("Player") then
            inGame = Players:FindFirstChild(owner.Name) ~= nil
        elseif type(owner) == "number" then
            inGame = Players:GetPlayerByUserId(owner) ~= nil
        elseif type(owner) == "table" and owner.Name then
            inGame = Players:FindFirstChild(tostring(owner.Name)) ~= nil
        elseif type(owner) == "string" then
            inGame = Players:FindFirstChild(owner) ~= nil
        elseif typeof(owner) == "Instance" and owner.Name then
            inGame = Players:FindFirstChild(owner.Name) ~= nil
        end
    end)
    return inGame
end

function getStealPromptForSlot(plot, slot)
    local podiums = plot and plot:FindFirstChild("AnimalPodiums")
    local podium = podiums and podiums:FindFirstChild(tostring(slot))
    local base = podium and podium:FindFirstChild("Base")
    local spawnp = base and base:FindFirstChild("Spawn")
    local att = spawnp and spawnp:FindFirstChild("PromptAttachment")
    local prompt = att and att:FindFirstChildWhichIsA("ProximityPrompt")
    if prompt and (prompt.ActionText:lower():find("steal") or prompt.ActionText == "") then return prompt end
    return nil
end

_BLOCKING_MACHINE_TYPES = {
    Fuse     = true,
    Duel     = true,
    Trade    = true,
    Crafting = true,
}
function _CartisIsFusing(animalData)
    if type(animalData) ~= "table" then return false end
    local m = animalData.Machine
    if type(m) ~= "table" then return false end
    return _BLOCKING_MACHINE_TYPES[m.Type] == true
end

function scanAllPets()
    local pets = {}
    if not loadModules() then return pets end

    local Plots = Workspace:FindFirstChild("Plots")
    if not Plots then return pets end

    for _, plot in ipairs(Plots:GetChildren()) do
        local channel = getPlotChannel(plot.Name)
        if not channel then continue end
        if isMyPlot(channel) then continue end
        -- if not ownerInGame(channel) then continue end

        local animalList = channelGet(channel, "AnimalList")
        if not animalList then continue end

        for slot, animalData in pairs(animalList) do
            if type(animalData) ~= "table" then continue end
            local animalName = animalData.Index
            if not animalName then continue end
            local animalInfo = AnimalsData and AnimalsData[animalName]
            if not animalInfo then continue end
            if _CartisIsFusing(animalData) then continue end

            local mutation = animalData.Mutation or "None"
            local genValue = 0
            pcall(function()
                genValue = AnimalsShared:GetGeneration(animalName, animalData.Mutation, animalData.Traits, nil)
            end)

            local displayName = (animalInfo and animalInfo.DisplayName) or animalName
            local pos = getPetPosition(plot, slot)

            if pos then
                table.insert(pets, {
                    name = displayName,
                    mps = genValue,
                    mutation = mutation,
                    position = pos,
                    plot = plot.Name,
                    slot = tostring(slot),
                    prompt = getStealPromptForSlot(plot, slot),
                })
            end
        end
    end

    table.sort(pets, function(a, b)
        return petOutranks(a.name, b.name, a.mutation, b.mutation, a.mps, b.mps)
    end)

    return pets
end

function findClosest(petPos, coordTable, fromPos)
    local all = {}
    for skyKey, coords in pairs(coordTable) do
        for _, data in ipairs(coords) do
            local c = data.coord
            local d = (petPos.X - c.X)^2 + (petPos.Z - c.Z)^2
            all[#all + 1] = { data = data, key = skyKey, petD = d }
        end
    end
    table.sort(all, function(a, b) return a.petD < b.petD end)
    if #all == 0 then return nil, nil end
    if not fromPos then return all[1].data, all[1].key end

    local cands = { all[1] }
    if all[2] and math.abs(all[2].data.coord.X - all[1].data.coord.X) < 40 then
        cands[#cands + 1] = all[2]
    end
    local best, bestKey, bestDist = all[1].data, all[1].key, math.huge
    for _, e in ipairs(cands) do
        local c = e.data.coord
        local d = (fromPos.X - c.X)^2 + (fromPos.Z - c.Z)^2
        if d < bestDist then bestDist = d; best = e.data; bestKey = e.key end
    end
    return best, bestKey
end

-- =====================================================================
-- Path visualization helpers
-- =====================================================================
_vizParts = {}
_vizGen = 0
function clearViz()
    _vizGen = _vizGen + 1
        if _vizBallAnimConn then
            pcall(function() _vizBallAnimConn:Disconnect() end)
            _vizBallAnimConn = nil
        end
    local batch = {}
    for i, p in ipairs(_vizParts) do batch[i] = p end
    table.clear(_vizParts)
    -- straight Destroy (no fade-out tween + no per-part deferred closure): the
    -- old fade spun up one TweenService tween per segment on every wipe.
    for _, p in ipairs(batch) do
        if p and p.Parent then pcall(function() p:Destroy() end) end
    end
end
function _vizFadeIn(p, target)
    -- instant set (no tween): the fade-in was pure TweenService overhead that
    -- competed with the flight loop and dropped FPS mid-TP.
    p.Transparency = target or 0
end
function vizLine(a, b, color)
    local d = b - a
    if d.Magnitude < 0.05 then return end
    local p = Instance.new("Part")
    p.Name = "MynxxVizPath" -- protected from the FPS boost handlers
    p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CastShadow = false
    -- cylinder = rounded caps -> segments blend into each other (no square joints)
    p.Shape = Enum.PartType.Cylinder
    p.Material = Enum.Material.Neon; p.Color = color
    -- draw straight to final length/pose/transparency -- no grow-in tween, so
    -- no per-frame TweenService work fighting the flight loop (keeps FPS up mid-TP)
        -- Feixe azul grosso, com brilho e transparência moderada.
        p.Size = Vector3.new(d.Magnitude, 2.4, 2.4)
        p.CFrame = CFrame.new((a + b) / 2, b) * CFrame.Angles(0, math.rad(90), 0)
        p.Transparency = 0.28
    p.Parent = Workspace
    _vizParts[#_vizParts + 1] = p
        return p
end
function vizDot(pos, color, sz)
    local p = Instance.new("Part")
    p.Name = "MynxxVizPath"
    p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CastShadow = false
    p.Shape = Enum.PartType.Ball; p.Material = Enum.Material.Neon; p.Color = color
    p.Size = Vector3.new(sz, sz, sz)
    p.Position = pos
    _vizFadeIn(p, 0)
    p.Parent = Workspace
    _vizParts[#_vizParts + 1] = p
        return p
end
function vizPath(fromPos, waypoints)
end

isGrabbleTeleporting = false
local isTeleporting = false

function getPlotKey(plotName)
    if not plotName then return nil end
    local first, second = plotName:match("^Plot_([B-D])([1-4])$")
    if first and second then return first end
    local first2, second2 = plotName:match("^Plot([B-D])([1-4])$")
    if first2 and second2 then return first2 end
    local first3 = plotName:match("^([B-D])[1-4]$")
    if first3 then return first3 end
    return nil
end

SPEED = 200
ARRIVE = 3

function vZero(hrp)
    if hrp then hrp.AssemblyLinearVelocity = Vector3.zero; hrp.AssemblyAngularVelocity = Vector3.zero end
end

MAX_CLIMB = 60

-- (Reactive dodge removed -- it caused orbiting; also frees top-level locals:
-- Luau caps the main chunk at 200 local registers.)

function velMoveThrough(hrp, waypoints, speedOverride, allowJump, quickStart)
    if not hrp or not hrp.Parent or #waypoints == 0 then return end
    local _runSpeed = math.clamp(tonumber(speedOverride or Config.TpSettings.GrabbleTPSpeed or (_G.MynxxCarpetSpeed or CARPET_SPEED or 450)) or 450, 50, 900)
    vizPath(hrp.Position, waypoints)
    local wpIdx = 1
    local done = false
    local conn
    local function finish()
        if done then return end
        done = true
        if hrp and hrp.Parent then
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
            local _, y = hrp.CFrame:ToEulerAnglesYXZ()
            hrp.CFrame = CFrame.new(waypoints[#waypoints]) * CFrame.Angles(0, y, 0)
        end
        if conn then conn:Disconnect() end
        -- keep the path visible a moment so a failed TP shows WHERE it failed
        task.delay(4, clearViz)
    end
    local lastDist, stall = math.huge, 0
    local lastEquip = 0

    if quickStart then
        local _hp = RaycastParams.new()
        _hp.FilterType = Enum.RaycastFilterType.Exclude
        _hp.IgnoreWater = true
        local _skip = {}
        for _, pl in ipairs(Players:GetPlayers()) do
            if pl.Character then _skip[#_skip + 1] = pl.Character end
        end
        _hp.FilterDescendantsInstances = _skip
        for _ = 1, 3 do
            local target = waypoints[wpIdx]
            if not target then break end
            local flat = Vector3.new(target.X - hrp.Position.X, 0, target.Z - hrp.Position.Z)
            local mag = flat.Magnitude
            if mag < 1 then break end
            local nextPos = hrp.Position + flat.Unit * math.min(35, mag)
            local _hit = Workspace:Raycast(hrp.Position, nextPos - hrp.Position, _hp)
            if _hit and _hit.Instance and _hit.Instance.CanCollide then break end
            hrp.CFrame = (hrp.CFrame - hrp.CFrame.Position) + nextPos
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
            RunService.Heartbeat:Wait()
            if not hrp or not hrp.Parent then return end
        end
    end

    conn = RunService.Heartbeat:Connect(function()
        if not hrp or not hrp.Parent or done then
            if conn then conn:Disconnect() end
            return
        end
        if os.clock() - lastEquip >= 0.25 then
            equipCarpet()
            lastEquip = os.clock()
        end
        local target = waypoints[wpIdx]
        local diff = target - hrp.Position
        local mag = diff.Magnitude
        if mag < ARRIVE then
            wpIdx = wpIdx + 1
            if wpIdx > #waypoints then finish() return end
            lastDist, stall = math.huge, 0
            target = waypoints[wpIdx]
            diff = target - hrp.Position
            mag = diff.Magnitude
        end
        if mag > lastDist - 0.05 then stall = stall + 1 else stall = 0 end
        lastDist = mag
        if stall >= 18 then finish() return end
        if mag >= 0.1 then
            local dir = diff.Unit
            if allowJump and diff.Y > 5 and wpIdx < #waypoints then
                local hum = hrp.Parent and hrp.Parent:FindFirstChildOfClass("Humanoid")
                if hum then
                    local st = hum:GetState()
                    if st ~= Enum.HumanoidStateType.Jumping and st ~= Enum.HumanoidStateType.Freefall then
                        pcall(function() hum:ChangeState(Enum.HumanoidStateType.Jumping) end)
                        pcall(function() hum.Jump = true end)
                    end
                end
            end
            local _sp = _runSpeed
            local _vy = dir.Y * _sp
            if _vy > MAX_CLIMB then _vy = MAX_CLIMB end
            hrp.AssemblyLinearVelocity = Vector3.new(dir.X * _sp, _vy, dir.Z * _sp)
        end
    end)
    local totalDist = 0
    local prev = hrp.Position
    for _, wp in ipairs(waypoints) do
        totalDist = totalDist + (prev - wp).Magnitude
        prev = wp
    end
    local timeout = math.max(2, totalDist / _runSpeed + 1.25)
    local elapsed = 0
    while not done and elapsed < timeout do
        task.wait(0.05)
        elapsed = elapsed + 0.05
    end
    finish()
    vZero(hrp)
end

-- =====================================================================
-- Raycast / route-pulling helpers
-- =====================================================================
_DIRS = { Vector3.new(1,0,0), Vector3.new(-1,0,0), Vector3.new(0,0,1), Vector3.new(0,0,-1) }
_STRUCT = { ["structure base home"] = true, ["Wall"] = true, ["Floor"] = true, ["Roof"] = true }
_SKIP_NAME = { ["DeliveryHitbox"]=true, ["StealHitbox"]=true, ["LaserHitbox"]=true,
    ["AnimalTarget"]=true, ["Multiplier"]=true, ["Laser"]=true, ["Hitbox"]=true,
    ["Spawn"]=true, ["MainRoot"]=true, ["SecondFloor"]=true, ["ThirdFloor"]=true, ["Slope"]=true }
_MAP_CENTER = { minX = -458, maxX = -362, minZ = -40, maxZ = 185 }
_BYPASS_Z_NORTH, _BYPASS_Z_SOUTH = 205, -95
_BYPASS_X_WEST, _BYPASS_X_EAST = -525, -295
_CENTER_OBSTACLE_KEYWORDS = {
    "craftingmachine", "crafting", "forge", "fuse", "duel", "trade",
    "machine", "spin", "wheel", "event", "portal", "npc", "vendor",
    "futbol", "livematches", "live matches",
}

function _isMapObstacle(inst)
    if not inst then return false end
    local p = inst
    while p and p ~= Workspace do
        local nl = p.Name:lower()
        for _, kw in ipairs(_CENTER_OBSTACLE_KEYWORDS) do
            if nl:find(kw, 1, true) then return true end
        end
        p = p.Parent
    end
    return false
end

function _blocks(inst)
    if not inst then return false end
    if _SKIP_NAME[inst.Name] then return false end
    if _isMapObstacle(inst) then return true end
    if inst.CanCollide then return true end
    if _STRUCT[inst.Name] then return true end
    local s = inst.Size
    if s and math.max(s.X * s.Y, s.X * s.Z, s.Y * s.Z) > 150 then return true end
    return false
end

function _block(origin, target)
    local rp = RaycastParams.new()
    rp.FilterType = Enum.RaycastFilterType.Exclude
    rp.IgnoreWater = true
    local skip = {}
    for _, pl in ipairs(Players:GetPlayers()) do
        if pl.Character then skip[#skip + 1] = pl.Character end
    end
    local o = origin
    for _ = 1, 16 do
        rp.FilterDescendantsInstances = skip
        local d = target - o
        if d.Magnitude < 0.05 then return nil end
        local res = Workspace:Raycast(o, d, rp)
        if not res then return nil end
        if _blocks(res.Instance) then return res end
        skip[#skip + 1] = res.Instance
        o = res.Position + d.Unit * 0.3
    end
    return nil
end

function _clear(a, b) return _block(a, b) == nil end

function _clearDist(origin, dir, maxD)
    local res = _block(origin, origin + dir.Unit * maxD)
    if not res then return maxD end
    return (res.Position - origin).Magnitude
end

function _len(pts)
    local s, prev = 0, pts[1]
    for k = 2, #pts do s = s + (pts[k] - prev).Magnitude; prev = pts[k] end
    return s
end

function _pull(pts)
    if #pts <= 2 then return pts end
    local out = { pts[1] }
    local i = 1
    while i < #pts do
        local j = #pts
        while j > i + 1 and not _clear(out[#out], pts[j]) do j = j - 1 end
        out[#out + 1] = pts[j]
        i = j
    end
    return out
end

function _stages(toPos)
    local st = {}
    for _, dr in ipairs(_DIRS) do
        local cd = _clearDist(toPos, dr, 46)
        if cd >= 12 then st[#st + 1] = toPos + dr * math.min(cd - 5, 38) end
    end
    return st
end

function _routeClear(pts)
    for i = 1, #pts - 1 do
        if not _clear(pts[i], pts[i + 1]) then return false end
    end
    return true
end

function _peakY(pts)
    local m = -math.huge
    for _, p in ipairs(pts) do if p.Y > m then m = p.Y end end
    return m
end

function _starts(fromPos)
    local pts = { fromPos }
    if _block(fromPos, fromPos + Vector3.new(0, 40, 0)) then
        for _, dr in ipairs(_DIRS) do
            local cd = _clearDist(fromPos, dr, 40)
            if cd >= 12 then pts[#pts + 1] = fromPos + dr * math.min(cd - 5, 34) end
        end
    end
    return pts
end

function _candidates(sp, stage, toPos)
    local list = {}
    local function add(mid)
        if mid then list[#list + 1] = { sp, mid, stage, toPos }
        else list[#list + 1] = { sp, stage, toPos } end
    end
    add(nil)
    add(Vector3.new(stage.X, sp.Y, stage.Z))
    add(Vector3.new(sp.X, stage.Y, sp.Z))
    local dir = Vector3.new(stage.X - sp.X, 0, stage.Z - sp.Z)
    if dir.Magnitude > 0.1 then
        dir = dir.Unit
        local perp = Vector3.new(-dir.Z, 0, dir.X)
        for _, off in ipairs({ 20, -20, 40, -40 }) do
            add(sp + perp * off)
        end
    end
    return list
end

PathfindingService = game:GetService("PathfindingService")
_CLEARANCE = 6

function _clearWide(a, b)
    if not _clear(a, b) then return false end
    local d = Vector3.new(b.X - a.X, 0, b.Z - a.Z)
    if d.Magnitude < 0.1 then return true end
    local p = Vector3.new(-d.Z, 0, d.X).Unit * _CLEARANCE
    return _clear(a + p, b + p) and _clear(a - p, b - p)
end

function _pullWide(pts)
    if #pts <= 2 then return pts end
    local out = { pts[1] }
    local i = 1
    while i < #pts do
        local j = #pts
        while j > i + 1 and not _clearWide(out[#out], pts[j]) do j = j - 1 end
        out[#out + 1] = pts[j]
        i = j
    end
    return out
end

function _inCenterZone(x, z)
    return x >= _MAP_CENTER.minX and x <= _MAP_CENTER.maxX
        and z >= _MAP_CENTER.minZ and z <= _MAP_CENTER.maxZ
end
function _segmentCrossesCenter(a, b)
    if _inCenterZone(a.X, a.Z) or _inCenterZone(b.X, b.Z) then return true end
    for i = 1, 10 do
        local t = i / 11
        if _inCenterZone(a.X + (b.X - a.X) * t, a.Z + (b.Z - a.Z) * t) then return true end
    end
    return false
end
function _findBestCenterDetour(fromPos, toPos, y)
    local candidates = {
        { Vector3.new(fromPos.X, y, _BYPASS_Z_NORTH), Vector3.new(toPos.X, y, _BYPASS_Z_NORTH) },
        { Vector3.new(fromPos.X, y, _BYPASS_Z_SOUTH), Vector3.new(toPos.X, y, _BYPASS_Z_SOUTH) },
        { Vector3.new(_BYPASS_X_WEST, y, fromPos.Z), Vector3.new(_BYPASS_X_WEST, y, toPos.Z) },
        { Vector3.new(_BYPASS_X_EAST, y, fromPos.Z), Vector3.new(_BYPASS_X_EAST, y, toPos.Z) },
    }
    local best, bestLen = nil, math.huge
    for _, pair in ipairs(candidates) do
        local w1, w2 = pair[1], pair[2]
        if _clearWide(fromPos, w1) and _clearWide(w1, w2) and _clearWide(w2, toPos) then
            local len = (fromPos - w1).Magnitude + (w1 - w2).Magnitude + (w2 - toPos).Magnitude
            if len < bestLen then bestLen = len; best = { w1, w2 } end
        end
    end
    return best
end

function computeRoute(fromPos, toPos, facingDir)
    local centerPatch = nil
    if _segmentCrossesCenter(fromPos, toPos) and not _clearWide(fromPos, toPos) then
        centerPatch = _findBestCenterDetour(fromPos, toPos, fromPos.Y)
        if centerPatch and #centerPatch > 0 then fromPos = centerPatch[#centerPatch] end
    end
    if _clearWide(fromPos, toPos) then
        if centerPatch then
            local merged = { centerPatch[1] }
            for i = 2, #centerPatch do merged[#merged + 1] = centerPatch[i] end
            merged[#merged + 1] = toPos
            return merged
        end
        return { toPos }
    end
    -- OVER-THE-TOP first: climb, straight, drop -- beats the ground pathfinder
    -- snaking around shops/stands whenever the sky is clear.
    for _, lift in ipairs({ 12, 20, 32, 45 }) do
        local a = fromPos + Vector3.new(0, lift, 0)
        local b = Vector3.new(toPos.X, math.max(toPos.Y + 4, a.Y), toPos.Z)
        if _clearWide(fromPos, a) and _clearWide(a, b) and _clearWide(b, toPos) then
            if centerPatch then
                local merged = { centerPatch[1] }
                for i = 2, #centerPatch do merged[#merged + 1] = centerPatch[i] end
                merged[#merged + 1] = a; merged[#merged + 1] = b; merged[#merged + 1] = toPos
                return merged
            end
            return { a, b, toPos }
        end
    end
    local entry = facingDir and (toPos - facingDir * 14) or toPos
    local groundTo = Vector3.new(entry.X, fromPos.Y, entry.Z)
    local path = PathfindingService:CreatePath({
        AgentRadius = 8, AgentHeight = 5, AgentCanJump = true, AgentJumpHeight = 10, AgentMaxSlope = 89,
    })
    local FLOAT = 3
    local nav = { fromPos }
    local ok = pcall(function()
        path:ComputeAsync(Vector3.new(fromPos.X, fromPos.Y, fromPos.Z), groundTo)
    end)
    if ok and path.Status == Enum.PathStatus.Success then
        local last = fromPos
        for _, wp in ipairs(path:GetWaypoints()) do
            if (wp.Position - last).Magnitude >= 8 then
                nav[#nav + 1] = wp.Position + Vector3.new(0, FLOAT, 0)
                last = wp.Position
            end
        end
    end
    nav[#nav + 1] = entry + Vector3.new(0, FLOAT, 0)
    local route = _pullWide(nav)
    route[#route + 1] = toPos
    if centerPatch and #centerPatch > 0 then
        local merged = { centerPatch[1] }
        for i = 2, #centerPatch do merged[#merged + 1] = centerPatch[i] end
        for _, p in ipairs(route) do merged[#merged + 1] = p end
        return _pullWide(merged)
    end
    return route
end

-- =====================================================================
-- Tool & Clone Helpers
-- =====================================================================

function getPetPosition(plot, slot)
    local podiums = plot:FindFirstChild("AnimalPodiums")
    if not podiums then return nil end
    local podium = podiums:FindFirstChild(tostring(slot))
    if not podium then return nil end

    for _, desc in ipairs(podium:GetDescendants()) do
        if desc:IsA("Model") and desc.Name ~= "Claim" and desc.Name ~= "Base" and desc.Name ~= "Decorations" then
            local hasMesh = false
            for _, c in ipairs(desc:GetDescendants()) do
                if c:IsA("MeshPart") then hasMesh = true; break end
            end
            if hasMesh then
                local ok, cf = pcall(function() return desc:GetBoundingBox() end)
                if ok then return cf.Position end
            end
        end
    end

    local ok, cf = pcall(function() return podium:GetPivot() end)
    if ok then return cf.Position end
    return podium.Position
end

end

cloneref = cloneref or function(o) return o end
getinfo = debug.getinfo or getinfo
lp = cloneref(game:GetService("Players")).LocalPlayer

function Strip()
    local char = lp.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    if not getconnections or not getinfo then return end
    for _, sig in ipairs({"CFrame", "Position"}) do
        local signal = hrp:GetPropertyChangedSignal(sig)
        if signal then
            for _, c in ipairs(getconnections(signal)) do
                local f = c.Function
                if f and c.Enabled and getinfo(f) and getinfo(f).source == "=ReplicatedFirst.test" then
                    pcall(function() c:Disable() end)
                end
            end
        end
    end
end

task.spawn(function()
    while true do
        pcall(Strip)
        task.wait(0.5)
    end
end)

function runAutoSnipe()
    pcall(Strip)
    if CarpetState and CarpetState.enabled then
        setCarpetSpeed(false)
        Config["Carpet Speed"] = false
        saveConfig()
    end
    if _G._isTpMoving then
        return
    end

    -- ============================================================
    -- FRESH SCAN: rescan ALL plots inline before picking target.
    -- (Vanish Hub style - guarantees latest Synchronizer data)
    -- ============================================================
    local cache = {}
    do
        local _RS = game:GetService("ReplicatedStorage")
        local _Packages = _RS:FindFirstChild("Packages")
        local _Datas    = _RS:FindFirstChild("Datas")
        local _Shared   = _RS:FindFirstChild("Shared")
        local _Utils    = _RS:FindFirstChild("Utils")
        local _Sync, _AData, _AShared, _NU
        pcall(function() _Sync   = _Packages and require(_Packages:FindFirstChild("Synchronizer")) end)
        pcall(function() _AData  = _Datas    and require(_Datas:FindFirstChild("Animals")) end)
        pcall(function() _AShared = _Shared  and require(_Shared:FindFirstChild("Animals")) end)
        pcall(function() _NU     = _Utils    and require(_Utils:FindFirstChild("NumberUtils")) end)

        local plotsFolder = workspace:FindFirstChild("Plots")
        if plotsFolder and _Sync and _AData and _AShared then
            for _, plot in ipairs(plotsFolder:GetChildren()) do
                pcall(function()
                    local data = _G.VanishGetSyncData and _G.VanishGetSyncData(plot.Name)
                    if not data then return end
                    local owner = data.Owner
                    local ownerName
                    if typeof(owner) == "Instance" and owner:IsA("Player") then ownerName = owner.Name
                    elseif type(owner) == "table" and owner.Name then ownerName = owner.Name
                    elseif type(owner) == "string" then ownerName = owner end
                    if not ownerName or ownerName == LocalPlayer.Name or not Players:FindFirstChild(ownerName) then return end
                    if _G.VanishIsDuelBlocking and _G.VanishIsDuelBlocking(ownerName) then return end
                    local al = data.AnimalList
                    if type(al) ~= "table" then return end
                    for slot, ad in pairs(al) do
                        if type(ad) == "table" and ad.Index then
                            if _G.VanishIsFusing and _G.VanishIsFusing(ad) then continue end
                            local info = _AData[ad.Index]
                            if info then
                                local gv = 0
                                pcall(function()
                                    gv = _AShared:GetGeneration(ad.Index, ad.Mutation, ad.Traits, nil) or 0
                                end)
                                if type(gv) == "number" and gv >= 1 then
                                    local genText
                                    if _NU then
                                        pcall(function() genText = "$" .. _NU:ToString(gv) .. "/s" end)
                                    end
                                    table.insert(cache, {
                                        name = info.DisplayName or ad.Index,
                                        Index = ad.Index,
                                        genText = genText or ("$" .. tostring(math.floor(gv)) .. "/s"),
                                        genValue = gv,
                                        mutation = ad.Mutation or "None",
                                        owner = ownerName,
                                        plot = plot.Name,
                                        slot = tostring(slot),
                                        uid = plot.Name .. "_" .. tostring(slot),
                                    })
                                end
                            end
                        end
                    end
                end)
            end
        end
        table.sort(cache, function(a, b) return a.genValue > b.genValue end)
    end

    local targetPetData = nil

    -- Target selection (Vanish Hub logic)
    if Config.AutoTPPriority then
        local bestEntry = nil
        if cache and type(cache) == "table" then
            for _, pName in ipairs(PRIORITY_LIST or {}) do
                local searchName = pName:lower()
                local bestForName, bestVal = nil, -math.huge
                for _, a in ipairs(cache) do
                    if a and a.name and a.name:lower() == searchName and a.owner ~= LocalPlayer.Name
                        and type(a.genValue) == "number" and a.genValue > bestVal then
                        bestForName = a; bestVal = a.genValue
                    end
                end
                if bestForName then bestEntry = bestForName; break end
            end
            if not bestEntry then
                bestEntry = cache[1]
            end
        end
        if bestEntry then
            targetPetData = bestEntry
        else
            if not SharedState.SelectedPetData then return end
            targetPetData = SharedState.SelectedPetData.animalData
        end
    else
        targetPetData = cache[1]
        if not targetPetData then
            if not SharedState.SelectedPetData then return end
            targetPetData = SharedState.SelectedPetData.animalData
        end
    end
    if not targetPetData then return end

    -- Pass target to grabble TP via manuallySelectedUID
    manuallySelectedUID = targetPetData.uid

    -- Use grabble/velocity TP
    _G._isTpMoving = true
    local fn = _G.MynxxStartSideTP or doGrabbleVelocityTP
    local okGrab, errGrab = pcall(fn)
    _G._isTpMoving = false
    manuallySelectedUID = nil
    if not okGrab then
        warn("Grabble TP error:", errGrab)
    end
end

function tpToBrainrot()
    local targetPetData = getTargetPetData()
    if not targetPetData then
        return
    end
    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    local hum = char and char:FindFirstChild("Humanoid")
    if not hrp or not hum or (hum.Health <= 0) then
        return
    end
    local targetPart = nil
    for _ = 1, 10 do
        targetPart = findAdorneeGlobal(targetPetData)
        if targetPart then
            break
        end
        task.wait(0.05)
    end
    if not targetPart then
        return
    end
    local ok, err = pcall(function()
        local carpetName = Config.TpSettings.Tool
        local carpet = LocalPlayer.Backpack:FindFirstChild(carpetName) or char:FindFirstChild(carpetName)
        if carpet then
            hum:EquipTool(carpet)
        end
        local targetPos = targetPart.Position
        local platY = getGoToBrainrotTargetY(targetPos.Y) - 1.5
        local plat = Instance.new("Part")
        plat.Name = "XiTempPlatform"
        plat.Size = Vector3.new(6, 1.5, 6)
        plat.Position = Vector3.new(targetPos.X, platY, targetPos.Z)
        plat.Color = Color3.fromRGB(232, 111, 177)
        plat.Material = Enum.Material.Neon
        plat.Anchored = true
        plat.CanCollide = false; pcall(makeOneWay, plat)
        plat.Transparency = 0.3
        plat.Parent = Workspace

        local targetCF = CFrame.new(targetPos.X, platY + 7, targetPos.Z)
        hrp.AssemblyLinearVelocity = Vector3.zero
        hrp.AssemblyAngularVelocity = Vector3.zero
        hrp.CFrame = targetCF
        task.wait(0.05)
        hrp.AssemblyLinearVelocity = Vector3.zero

        task.spawn(function()
            local start = tick()
            while (tick() - start) < 20 do
                if LocalPlayer:GetAttribute("Stealing") then
                    break
                end
                task.wait(0.1)
            end
            plat:Destroy()
        end)
    end)
    if not ok then
        warn("tpToBrainrot error:", err)
    end
end
_G.runAutoSnipe = runAutoSnipe
_G.tpToBrainrot = tpToBrainrot

-- ============================================================
-- CLICK TO AP
-- ============================================================
ctapHighlight=Instance.new("Highlight",CoreGui)
ctapHighlight.FillColor=Color3.fromRGB(255,150,200); ctapHighlight.FillTransparency=0.3
ctapHighlight.OutlineColor=Color3.fromRGB(255,150,200); ctapHighlight.OutlineTransparency=0
ctapHighlight.Adornee=nil; ctapHighlight.DepthMode=Enum.HighlightDepthMode.AlwaysOnTop

function rayToCubeIntersect(rayOrigin,rayDirection,cubeCenter,cubeSize)
    local halfSize=cubeSize/2; local minB=cubeCenter-Vector3.new(halfSize,halfSize,halfSize); local maxB=cubeCenter+Vector3.new(halfSize,halfSize,halfSize)
    local rd=Vector3.new(rayDirection.X==0 and 0.0001 or rayDirection.X, rayDirection.Y==0 and 0.0001 or rayDirection.Y, rayDirection.Z==0 and 0.0001 or rayDirection.Z)
    local tmin,tmax=(minB.X-rayOrigin.X)/rd.X,(maxB.X-rayOrigin.X)/rd.X; if tmin>tmax then tmin,tmax=tmax,tmin end
    local tymin,tymax=(minB.Y-rayOrigin.Y)/rd.Y,(maxB.Y-rayOrigin.Y)/rd.Y; if tymin>tymax then tymin,tymax=tymax,tymin end
    if tmin>tymax or tymin>tmax then return false end; if tymin>tmin then tmin=tymin end; if tymax<tmax then tmax=tymax end
    local tzmin,tzmax=(minB.Z-rayOrigin.Z)/rd.Z,(maxB.Z-rayOrigin.Z)/rd.Z; if tzmin>tzmax then tzmin,tzmax=tzmax,tzmin end
    return not(tmin>tzmax or tzmin>tmax)
end

RunService.RenderStepped:Connect(function()
    if Config.ClickToAP then
        local camera=Workspace.CurrentCamera; local mousePos=UIS:GetMouseLocation()
        local ray=camera:ViewportPointToRay(mousePos.X,mousePos.Y); local bestPlayer,bestDist=nil,math.huge
        for _,p in ipairs(Players:GetPlayers()) do if p~=LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            if rayToCubeIntersect(ray.Origin,ray.Direction,p.Character.HumanoidRootPart.Position,Config.ClickToAPRadius or 8) then
                local dist=(ray.Origin-p.Character.HumanoidRootPart.Position).Magnitude; if dist<bestDist then bestDist=dist; bestPlayer=p end
            end
        end end
        ctapHighlight.Adornee=(bestPlayer and bestPlayer.Character) or nil
    else ctapHighlight.Adornee=nil end
end)

UIS.InputBegan:Connect(function(inp,g)
    if not g and inp.UserInputType==Enum.UserInputType.MouseButton1 and Config.ClickToAP then
        local camera=Workspace.CurrentCamera; local mousePos=UIS:GetMouseLocation()
        local ray=camera:ViewportPointToRay(mousePos.X,mousePos.Y); local bestPlayer,bestDist=nil,math.huge
        for _,p in ipairs(Players:GetPlayers()) do if p~=LocalPlayer and p.Character and p.Character:FindFirstChild("HumanoidRootPart") then
            if rayToCubeIntersect(ray.Origin,ray.Direction,p.Character.HumanoidRootPart.Position,Config.ClickToAPRadius or 8) then
                local dist=(ray.Origin-p.Character.HumanoidRootPart.Position).Magnitude; if dist<bestDist then bestDist=dist; bestPlayer=p end
            end
        end end
        if bestPlayer then
            if isPlayerBlacklisted(bestPlayer) then
                ShowNotification("CLICK AP", bestPlayer.DisplayName .. " is blacklisted")
                return
            end
            local cmdList = Config.ClickToAPOrder or AP_ALL_COMMANDS
            local startIndex = _G.ClickToAPIndex or 1
            if startIndex > #cmdList then startIndex = 1 end
            
            local picked = nil
            local attempts = 0
            while attempts < #cmdList do
                local idx = ((startIndex - 1 + attempts) % #cmdList) + 1
                local cmd = cmdList[idx]
                if Config.ClickToAPCommands and Config.ClickToAPCommands[cmd] and not apIsOnCooldown(cmd) then
                    picked = cmd
                    _G.ClickToAPIndex = idx + 1
                    break
                end
                attempts = attempts + 1
            end

            if not picked then
                ShowNotification("CLICK AP", "No commands enabled or all on cooldown")
                return
            end

            if runAdminCommand(bestPlayer, picked) then
                local emoji = AP_COMMAND_EMOJIS[picked] or "⚡"
                ShowNotification("CLICK AP", emoji .. " " .. picked .. " → " .. bestPlayer.DisplayName)
            end
        end
    end
end)

-- ============================================================
-- ESP
-- ============================================================
-- Player ESP
playerESPEnabled=Config.PlayerESP; local playerBillboards={}
DANGER_TOOLS={["Boogie Bomb"]=true,["Medusa's Head"]=true,["Body Swap Potion"]=true,["Laser Cape"]=true,["Rainbowrath Sword"]=true,["Gummy Bear"]=true}
function getHeldTool(p) local c=p.Character; if not c then return nil end; for _,o in ipairs(c:GetChildren()) do if o:IsA("Tool") then return o.Name end end; return nil end
function makePlayerBillboard(plr)
    local bb=Instance.new("BillboardGui"); bb.Name="PlayerESP_"..tostring(plr.UserId); bb.Size=UDim2.new(0,170,0,48)
    bb.StudsOffsetWorldSpace=Vector3.new(0,2.8,0); bb.AlwaysOnTop=true; bb.LightInfluence=0; bb.ResetOnSpawn=false
    local nameLbl=Instance.new("TextLabel",bb); nameLbl.Size=UDim2.new(1,0,0,18); nameLbl.BackgroundTransparency=1
    nameLbl.Font=Enum.Font.GothamBold; nameLbl.TextSize=14; nameLbl.TextColor3=Color3.fromRGB(255,255,255)
    nameLbl.TextStrokeTransparency=0.4; nameLbl.TextStrokeColor3=Color3.fromRGB(0,0,0); nameLbl.Text=plr.Name
    local toolLbl=Instance.new("TextLabel",bb); toolLbl.Name="ToolLabel"; toolLbl.Size=UDim2.new(1,0,0,13); toolLbl.Position=UDim2.new(0,0,0,18)
    toolLbl.BackgroundTransparency=1; toolLbl.Font=Enum.Font.GothamMedium; toolLbl.TextSize=11; toolLbl.TextColor3=Color3.fromRGB(100,220,255)
    toolLbl.TextStrokeTransparency=0.4; toolLbl.TextStrokeColor3=Color3.fromRGB(0,0,0); toolLbl.Text=getHeldTool(plr) or ""
    local stealLbl=Instance.new("TextLabel",bb); stealLbl.Name="StealLabel"; stealLbl.Size=UDim2.new(1,0,0,13); stealLbl.Position=UDim2.new(0,0,0,31)
    stealLbl.BackgroundTransparency=1; stealLbl.Font=Enum.Font.GothamBold; stealLbl.TextSize=11; stealLbl.TextColor3=Color3.fromRGB(255,60,60)
    stealLbl.TextStrokeTransparency=0.4; stealLbl.TextStrokeColor3=Color3.fromRGB(0,0,0); stealLbl.Text=""
    return bb,nameLbl
end
function createOrRefreshPlayerESP(plr)
    if plr==LocalPlayer then return end; local hrp=plr.Character and plr.Character:FindFirstChild("HumanoidRootPart"); if not hrp then return end
    local hum=plr.Character:FindFirstChild("Humanoid"); if hum then hum.DisplayDistanceType=Enum.HumanoidDisplayDistanceType.None end
    local uid=plr.UserId; local entry=playerBillboards[uid]
    if not entry or not entry.bb or not entry.bb.Parent then
        if entry and entry.bb then pcall(function() entry.bb:Destroy() end) end
        local bb,nameLbl=makePlayerBillboard(plr); bb.Adornee=hrp; bb.Parent=hrp; playerBillboards[uid]={bb=bb,nameLbl=nameLbl,player=plr}
    elseif entry.bb.Adornee~=hrp then entry.bb.Adornee=hrp; entry.bb.Parent=hrp end
end
function clearPlayerESP()
    for uid,entry in pairs(playerBillboards) do if entry.bb then pcall(entry.bb.Destroy,entry.bb) end; playerBillboards[uid]=nil end
end
task.spawn(function() while true do task.wait(0.5)
    if _G._isTpMoving then continue end -- ESP frozen during a TP (kept hidden by the TP-focus controller); don't rebuild
    if playerESPEnabled then
        for _,plr in ipairs(Players:GetPlayers()) do if plr~=LocalPlayer then pcall(createOrRefreshPlayerESP,plr) end end
        for uid,entry in pairs(playerBillboards) do if entry.bb and entry.bb.Parent then
            pcall(function() local tl=entry.bb:FindFirstChild("ToolLabel"); if tl then local ht=getHeldTool(entry.player); tl.Text=ht or ""
                if entry.nameLbl then entry.nameLbl.TextColor3=ht and DANGER_TOOLS[ht] and Color3.fromRGB(255,60,60) or Color3.fromRGB(255,255,255) end
            end end)
            pcall(function() local sl=entry.bb:FindFirstChild("StealLabel"); if sl then
                if entry.player:GetAttribute("Stealing") then
                    -- always re-resolve: getStealingInfo locks the name per
                    -- steal session, so this is cheap AND self-corrects if an
                    -- early generic/wrong name was shown
                    local _, an = getStealingInfo(entry.player)
                    if an then entry.stealName = an end
                    sl.Text = "Stealing : "..tostring(entry.stealName or "Brainrot")
                else
                    entry.stealName = nil
                    sl.Text = ""
                end
            end end)
        end end
    else clearPlayerESP() end
end end)

-- ANTI-INVISIBILITY: some hubs/potions make players semi-transparent (you see
-- them "less than the others"). Force their characters fully visible locally.
-- Purely visual -- the AP protection on hub users is untouched.
task.spawn(function()
    while true do task.wait(0.5)
        -- PERF: skip during TPs / first 10s after execute (FPS-critical)
        if _G._isTpMoving or (os.clock() - (_G.__MynxxBootClock or 0)) < 2 then continue end
        pcall(function()
            for _, plr in ipairs(Players:GetPlayers()) do
                if plr ~= LocalPlayer and plr.Character then
                    for _, d in ipairs(plr.Character:GetDescendants()) do
                        if d:IsA("BasePart") and d.Name ~= "HumanoidRootPart" then
                            if d.Transparency > 0.05 and d.Transparency < 0.95 then d.Transparency = 0 end
                            if d.LocalTransparencyModifier > 0.05 then d.LocalTransparencyModifier = 0 end
                        elseif (d:IsA("Decal") or d:IsA("Texture")) and d.Transparency > 0.05 and d.Transparency < 0.95 then
                            d.Transparency = 0
                        end
                    end
                end
            end
        end)
    end
end)

-- BASE STEALING TAG: while a player is stealing, shows "Stealing : <brainrot>"
-- in red, floating over THEIR OWN base. (do-block: keeps locals off the main
-- chunk's 200-register cap)
do
    local tags = {} -- [userId] = { bb = BillboardGui }
    local function plotOfPlayer(plr)
        local plots = Workspace:FindFirstChild("Plots")
        if not plots then return nil end
        for _, plot in ipairs(plots:GetChildren()) do
            if getPlotOwner(plot) == plr then return plot end
        end
        return nil
    end
    local function anchorPart(plot)
        local sign = plot:FindFirstChild("PlotSign")
        if sign then
            if sign:IsA("BasePart") then return sign end
            local p = sign:FindFirstChildWhichIsA("BasePart", true)
            if p then return p end
        end
        return plot:FindFirstChildWhichIsA("BasePart", true)
    end
    local function clearTag(uid)
        local e = tags[uid]
        if e then pcall(function() e.bb:Destroy() end); tags[uid] = nil end
    end
    task.spawn(function()
        while true do
            task.wait(0.5)
            -- PERF: skip during TPs / first 10s after execute (FPS-critical)
            if _G._isTpMoving or (os.clock() - (_G.__MynxxBootClock or 0)) < 2 then continue end
            for _, plr in ipairs(Players:GetPlayers()) do
                if plr ~= player then
                    local uid = plr.UserId
                    if plr:GetAttribute("Stealing") then
                        local e = tags[uid]
                        if not e or not e.bb or not e.bb.Parent then
                            clearTag(uid)
                            pcall(function()
                                local plot = plotOfPlayer(plr)
                                local part = plot and anchorPart(plot)
                                if not part then return end
                                local _, petName = getStealingInfo(plr)
                                local bb = Instance.new("BillboardGui")
                                bb.Name = "MynxxBaseStealTag_" .. uid
                                bb.Size = UDim2.new(0, 340, 0, 46)
                                bb.StudsOffsetWorldSpace = Vector3.new(0, 14, 0)
                                bb.AlwaysOnTop = true
                                bb.MaxDistance = 2000
                                bb.LightInfluence = 0
                                local lbl = Instance.new("TextLabel", bb)
                                lbl.Size = UDim2.new(1, 0, 1, 0)
                                lbl.BackgroundTransparency = 1
                                lbl.Font = Enum.Font.GothamBlack
                                lbl.TextSize = 24
                                lbl.TextColor3 = Color3.fromRGB(255, 40, 40)
                                lbl.TextStrokeTransparency = 0.15
                                lbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
                                lbl.Text = "Stealing : " .. tostring(petName or "Brainrot")
                                bb.Adornee = part
                                bb.Parent = part
                                tags[uid] = { bb = bb, lbl = lbl }
                            end)
                        elseif e.lbl then
                            -- always re-resolve: the per-steal session lock in
                            -- getStealingInfo makes this cheap and corrects an
                            -- early generic/wrong name (e.g. morph model)
                            pcall(function()
                                local _, petName = getStealingInfo(plr)
                                if petName then
                                    e.lbl.Text = "Stealing : " .. tostring(petName)
                                end
                            end)
                        end
                    else
                        clearTag(uid)
                    end
                end
            end
            for uid in pairs(tags) do
                if not Players:GetPlayerByUserId(uid) then clearTag(uid) end
            end
        end
    end)
end

-- STEAL ALERT HUD: topo central, achatado, com animação de aparecer/sumir
do
    local _alertParent = (gethui and gethui()) or game:GetService("CoreGui")
    local oldSg = _alertParent:FindFirstChild("MynxxStealAlert")
    if oldSg then oldSg:Destroy() end
    local stealAlertSg = Instance.new("ScreenGui")
    stealAlertSg.Name = "MynxxStealAlert"
    stealAlertSg.ResetOnSpawn = false
    stealAlertSg.IgnoreGuiInset = true
    stealAlertSg.DisplayOrder = 9999998
    stealAlertSg.Parent = _alertParent

    local frame = Instance.new("Frame")
    frame.Size = UDim2.new(0, 280, 0, 75)
    frame.Position = UDim2.new(0.5, -140, 0, -100)
    frame.BackgroundColor3 = Color3.fromRGB(10, 10, 12)
    frame.BackgroundTransparency = 0.2
    frame.BorderSizePixel = 0
    frame.Parent = stealAlertSg
    frame.ClipsDescendants = true
    local fc = Instance.new("UICorner"); fc.CornerRadius = UDim.new(0, 10); fc.Parent = frame
    local fs = Instance.new("UIStroke"); fs.Color = Color3.fromRGB(255, 40, 40); fs.Thickness = 2.5; fs.Parent = frame

    local avatarFrame = Instance.new("Frame")
    avatarFrame.Size = UDim2.new(0, 54, 0, 54)
    avatarFrame.Position = UDim2.new(0, 10, 0.5, -27)
    avatarFrame.BackgroundColor3 = Color3.fromRGB(20, 20, 22)
    avatarFrame.BackgroundTransparency = 0.2
    avatarFrame.BorderSizePixel = 0
    avatarFrame.Parent = frame
    local avCorner = Instance.new("UICorner"); avCorner.CornerRadius = UDim.new(0, 8); avCorner.Parent = avatarFrame
    local avatarImg = Instance.new("ImageLabel")
    avatarImg.Size = UDim2.new(1, -4, 1, -4)
    avatarImg.Position = UDim2.new(0, 2, 0, 2)
    avatarImg.BackgroundTransparency = 1
    avatarImg.Image = ""
    avatarImg.Parent = avatarFrame
    local avImgCorner = Instance.new("UICorner"); avImgCorner.CornerRadius = UDim.new(0, 6); avImgCorner.Parent = avatarImg

    local nameLbl = Instance.new("TextLabel")
    nameLbl.Size = UDim2.new(1, -78, 0, 18)
    nameLbl.Position = UDim2.new(0, 72, 0, 14)
    nameLbl.BackgroundTransparency = 1
    nameLbl.Font = Enum.Font.GothamBlack
    nameLbl.TextSize = 13
    nameLbl.TextColor3 = Color3.fromRGB(255, 255, 255)
    nameLbl.Text = ""
    nameLbl.TextXAlignment = Enum.TextXAlignment.Left
    nameLbl.TextTruncate = Enum.TextTruncate.AtEnd
    nameLbl.Parent = frame

    local petLbl = Instance.new("TextLabel")
    petLbl.Size = UDim2.new(1, -78, 0, 14)
    petLbl.Position = UDim2.new(0, 72, 0, 35)
    petLbl.BackgroundTransparency = 1
    petLbl.Font = Enum.Font.GothamMedium
    petLbl.TextSize = 10
    petLbl.TextColor3 = Color3.fromRGB(200, 200, 200)
    petLbl.Text = ""
    petLbl.TextXAlignment = Enum.TextXAlignment.Left
    petLbl.TextTruncate = Enum.TextTruncate.AtEnd
    petLbl.Parent = frame

    local valLbl = Instance.new("TextLabel")
    valLbl.Size = UDim2.new(1, -78, 0, 14)
    valLbl.Position = UDim2.new(0, 72, 0, 52)
    valLbl.BackgroundTransparency = 1
    valLbl.Font = Enum.Font.GothamMedium
    valLbl.TextSize = 10
    valLbl.TextColor3 = Color3.fromRGB(56, 214, 110)
    valLbl.Text = ""
    valLbl.TextXAlignment = Enum.TextXAlignment.Left
    valLbl.RichText = true
    valLbl.Visible = true
    valLbl.Parent = frame



    frame.Visible = false

    local alertEntry = nil
    local alertOscT = 0
    local alertRotConn
    local hideTween, showTween
    local _hideCompleteConn
    local _currentAlertPlr = nil

    local function cleanupAlertClone()
        if alertEntry then
            if alertEntry.watcher then task.cancel(alertEntry.watcher) end
            if alertEntry.clone then pcall(function() alertEntry.clone:Destroy() end) end
            alertEntry = nil
        end
    end

    local function _alertFindModel(animalData)
        if not animalData or not animalData.plot or not animalData.slot then return nil end
        local plots = Workspace:FindFirstChild("Plots")
        local plot = plots and plots:FindFirstChild(animalData.plot)
        if not plot then return nil end
        local podiums = plot:FindFirstChild("AnimalPodiums")
        local podium = podiums and podiums:FindFirstChild(animalData.slot)
        if not podium then return nil end
        local base = podium:FindFirstChild("Base")
        local spawnPart = base and (base:FindFirstChild("Spawn") or base:FindFirstChildWhichIsA("BasePart") or base)
        if not spawnPart then return nil end
        local spawnPos = spawnPart.Position
        local bestModel, bestScore = nil, -1
        for _, obj in ipairs(plot:GetDescendants()) do
            if not obj:IsA("Model") or obj.Name == "Base" then continue end
            local root = obj:FindFirstChild("HumanoidRootPart") or obj.PrimaryPart or obj:FindFirstChildWhichIsA("BasePart")
            if not root then continue end
            local dist = (root.Position - spawnPos).Magnitude
            if dist > 10 then continue end
            local score = -dist
            local idx = animalData.animalIndex or animalData.index or ""
            if idx ~= "" and (obj.Name == idx or obj.Name:find(idx, 1, true)) then score = score + 500 end
            if animalData.name and (obj.Name == animalData.name or obj.Name:find(animalData.name, 1, true)) then score = score + 300 end
            if obj:FindFirstChildOfClass("Humanoid") then score = score + 50 end
            if score > bestScore then bestScore = score; bestModel = obj end
        end
        return bestModel
    end

    local function _alertTryClone(entry, animalData)
        if not animalData then return false end
        local sourceModel = _alertFindModel(animalData)
        if not sourceModel then return false end
        pcall(function()
            local c = sourceModel:Clone()
            for _, d in ipairs(c:GetDescendants()) do
                if d:IsA("Script") or d:IsA("LocalScript") or d:IsA("ModuleScript") or d:IsA("Sound") or d:IsA("SoundGroup") or d:IsA("BodyMover") or d:IsA("Constraint") then
                    d:Destroy()
                elseif d:IsA("BasePart") then
                    d.Anchored = true; d.CanCollide = false
                elseif d:IsA("Animator") then
                    for _, tr in ipairs(d:GetPlayingAnimationTracks()) do tr:Stop(0) end
                end
            end
            c.Parent = vpFrame
            entry.clone = c
            local _, sz = c:GetBoundingBox()
            local r = math.max(math.max(sz.X, sz.Y, sz.Z) * 0.55 + 0.5, 2.5)
            vpCam.CFrame = CFrame.new(r * 2.8, r * 0.4, r * 2.8, 0, 0, -1, 0, 1, 0, 1, 0, 0)
        end)
        return entry.clone and entry.clone.Parent or false
    end

    local function _alertTryCloneAnywhere(plr, petName)
        if not petName then return false end
        return pcall(function()
            local lowerPet = petName:lower()
            local names = _MynxxAnimalNames
            local animalIndex
            if names then
                for k, v in pairs(names) do
                    if v:lower() == lowerPet then animalIndex = k; break end
                end
            end
            local function tryCloneFromModel(sourceModel)
                if not sourceModel or not sourceModel:IsA("Model") then return false end
                local c = sourceModel:Clone()
                local hasPart
                for _, d in ipairs(c:GetDescendants()) do
                    if d:IsA("Script") or d:IsA("LocalScript") or d:IsA("ModuleScript") or d:IsA("Sound") or d:IsA("SoundGroup") or d:IsA("BodyMover") or d:IsA("Constraint") or d:IsA("Weld") or d:IsA("Snap") or d:IsA("Motor6D") then
                        d:Destroy()
                    elseif d:IsA("BasePart") then
                        d.Anchored = true; d.CanCollide = false
                        d.CFrame = CFrame.new(0, 0, 0)
                        hasPart = true
                    elseif d:IsA("Animator") then
                        for _, tr in ipairs(d:GetPlayingAnimationTracks()) do tr:Stop(0) end
                    end
                end
                if hasPart then
                    c.Parent = vpFrame
                    local _, sz = c:GetBoundingBox()
                    local r = math.max(math.max(sz.X, sz.Y, sz.Z) * 0.55 + 0.5, 2.5)
                    vpCam.CFrame = CFrame.new(r * 2.8, r * 0.4, r * 2.8, 0, 0, -1, 0, 1, 0, 1, 0, 0)
                    return true
                end
                return false
            end
            local function matchModel(m)
                if not m:IsA("Model") or m.Name == "Base" then return false end
                if animalIndex and (m.Name == animalIndex or m.Name:lower() == animalIndex:lower()) then return true end
                if lowerPet ~= "" and (m.Name:lower() == lowerPet or m.Name:lower():find(lowerPet, 1, true)) then return true end
                if m:FindFirstChildOfClass("Humanoid") then return true end
                return false
            end
            local plots = Workspace:FindFirstChild("Plots")
            if not plots then return false end
            if plr then
                local thiefPlot, _ = getPlayerBaseInfo(plr)
                if thiefPlot then
                    for _, m in ipairs(thiefPlot:GetDescendants()) do
                        if matchModel(m) then
                            if tryCloneFromModel(m) then return true end
                        end
                    end
                end
            end
            for _, plot in ipairs(plots:GetChildren()) do
                for _, m in ipairs(plot:GetDescendants()) do
                    if matchModel(m) then
                        if tryCloneFromModel(m) then return true end
                    end
                end
            end
            return false
        end)
    end

    function _pbFmtVal(v)
        if not v or v == 0 then return "$0" end
        if v >= 1e12 then return "$" .. string.format("%.2fT", v / 1e12) end
        if v >= 1e9 then return "$" .. string.format("%.2fB", v / 1e9) end
        if v >= 1e6 then return "$" .. string.format("%.2fM", v / 1e6) end
        if v >= 1e3 then return "$" .. string.format("%.2fK", v / 1e3) end
        return "$" .. string.format("%.0f", v)
    end

    local function showStealAlert(plr, petName)
        if showTween then showTween:Cancel() end
        if hideTween then hideTween:Cancel() end
        if _hideCompleteConn then _hideCompleteConn:Disconnect(); _hideCompleteConn = nil end
        _currentAlertPlr = plr
        nameLbl.Text = "@" .. (plr.DisplayName or plr.Name)
        petLbl.Text = petName or "Brainrot"
        valLbl.Text = ""
        cleanupAlertClone()

        local animalData
        local cache = SharedState and SharedState.AllAnimalsCache
        if cache and petName then
            for _, a in ipairs(cache) do
                if a.name == petName or a.index == petName or (a.animalIndex and a.animalIndex == petName) then
                    animalData = a
                    break
                end
            end
            if not animalData then
                for _, a in ipairs(cache) do
                    if petName and (a.name:find(petName, 1, true) or (a.index and a.index:find(petName, 1, true))) then
                        animalData = a
                        break
                    end
                end
            end
            if not animalData and #cache > 0 then
                animalData = cache[1]
            end
        end

        local hasGen = animalData and animalData.genValue and animalData.genValue > 0
        if hasGen then
            valLbl.Text = '<font color="#38D66E">Gen:</font> ' .. _pbFmtVal(animalData.genValue) .. '/s'
        end
        valLbl.Visible = hasGen

        avatarImg.Image = ""
        task.spawn(function()
            local ok, url = pcall(function()
                return Players:GetUserThumbnailAsync(plr.UserId, Enum.ThumbnailType.HeadShot, Enum.ThumbnailSize.Size48x48)
            end)
            if ok and url ~= "" then
                avatarImg.Image = url
            end
        end)

        frame.Position = UDim2.new(0.5, -140, 0, -120)
        frame.Visible = true
        frame.BackgroundTransparency = 1
        fs.Transparency = 1
        avatarFrame.BackgroundTransparency = 1
        nameLbl.TextTransparency = 1
        petLbl.TextTransparency = 1
        valLbl.TextTransparency = 1
        showTween = TweenService:Create(frame, TweenInfo.new(0.35, Enum.EasingStyle.Quad, Enum.EasingDirection.Out), {
            Position = UDim2.new(0.5, -140, 0, 12),
            BackgroundTransparency = 0.2
        })
        showTween:Play()
        TweenService:Create(fs, TweenInfo.new(0.35), {Transparency = 0}):Play()
        TweenService:Create(avatarFrame, TweenInfo.new(0.3), {BackgroundTransparency = 0.2}):Play()
        TweenService:Create(nameLbl, TweenInfo.new(0.3), {TextTransparency = 0}):Play()
        TweenService:Create(petLbl, TweenInfo.new(0.3), {TextTransparency = 0}):Play()
        TweenService:Create(valLbl, TweenInfo.new(0.3), {TextTransparency = 0}):Play()
    end

    local function hideStealAlert()
        if showTween then showTween:Cancel() end
        if hideTween then hideTween:Cancel() end
        if _hideCompleteConn then _hideCompleteConn:Disconnect() end
        hideTween = TweenService:Create(frame, TweenInfo.new(0.4, Enum.EasingStyle.Quad, Enum.EasingDirection.In), {
            Position = UDim2.new(0.5, -140, 0, -120),
            BackgroundTransparency = 1
        })
        hideTween:Play()
        TweenService:Create(fs, TweenInfo.new(0.3), {Transparency = 1}):Play()
        TweenService:Create(avatarFrame, TweenInfo.new(0.25), {BackgroundTransparency = 1}):Play()
        TweenService:Create(nameLbl, TweenInfo.new(0.25), {TextTransparency = 1}):Play()
        TweenService:Create(petLbl, TweenInfo.new(0.25), {TextTransparency = 1}):Play()
        TweenService:Create(valLbl, TweenInfo.new(0.25), {TextTransparency = 1}):Play()
        _hideCompleteConn = hideTween.Completed:Connect(function()
            frame.Visible = false
        end)
    end

    local _currentBestKey = nil
    task.spawn(function()
        while true do
            task.wait(0.5)
            local bestPlr, bestPet, bestVal = nil, nil, -1
            for _, plr in ipairs(Players:GetPlayers()) do
                if plr ~= player and plr:GetAttribute("Stealing") then
                    local _, petName = getStealingInfo(plr)
                    local val = 0
                    local cache = SharedState and SharedState.AllAnimalsCache
                    if cache and petName then
                        for _, a in ipairs(cache) do
                            if a.name == petName or a.index == petName or (a.animalIndex and a.animalIndex == petName) then
                                val = a.genValue or 0; break
                            end
                        end
                        if val == 0 then
                            for _, a in ipairs(cache) do
                                if petName and (a.name:find(petName, 1, true) or (a.index and a.index:find(petName, 1, true))) then
                                    val = a.genValue or 0; break
                                end
                            end
                        end
                    end

                    if val > bestVal then
                        bestVal = val; bestPlr = plr; bestPet = petName
                    end
                end
            end
            if bestPlr then
                local key = bestPlr.UserId .. "|" .. (bestPet or "")
                if _currentBestKey ~= key then
                    _currentBestKey = key
                    showStealAlert(bestPlr, bestPet)
                end
            else
                if _currentBestKey then
                    _currentBestKey = nil
                    hideStealAlert()
                end
            end
        end
    end)

end

-- Line To Base ESP (kinqs beam style)
do
    local BEAM_NAME  = "PlotBeam"
    local ATT0_NAME  = "PlotBeamAttach_Player"
    local ATT1_NAME  = "PlotBeamAttach_Plot"
    local BEAM_COLOR = Color3.fromRGB(255, 40, 40)

    local plotBeam = nil
    local plotAtt0 = nil
    local plotAtt1 = nil
    local currentPlotPart = nil

    local function applyPlotBeamStyle(beam)
        if not beam then return end
        beam.FaceCamera = true
        beam.LightEmission = 1
        beam.Color = ColorSequence.new(BEAM_COLOR)
        beam.Transparency = NumberSequence.new(0.35)
        beam.Width0 = 0.45
        beam.Width1 = 0.45
        beam.TextureMode = Enum.TextureMode.Wrap
        beam.TextureSpeed = 0
        beam.Enabled = true
    end

    local function destroyPlotBeam()
        if plotBeam then pcall(function() plotBeam:Destroy() end) end
        local glow = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
        if glow then
            local old = glow:FindFirstChild("PlotBeamGlow")
            if old then pcall(function() old:Destroy() end) end
        end
        if plotAtt0 then pcall(function() plotAtt0:Destroy() end) end
        if plotAtt1 then pcall(function() plotAtt1:Destroy() end) end
        plotBeam, plotAtt0, plotAtt1 = nil, nil, nil
        currentPlotPart = nil
    end

    local function findMyPlot()
        local plots = Workspace:FindFirstChild("Plots")
        if not plots then return nil end
        for _, plot in ipairs(plots:GetChildren()) do
            if _G.isMyPlot_Instant and _G.isMyPlot_Instant(plot.Name) then
                return plot
            end
        end
        for _, plot in ipairs(plots:GetChildren()) do
            local sign = plot:FindFirstChild("PlotSign")
            if sign then
                local yb = sign:FindFirstChild("YourBase")
                if yb and yb:IsA("BillboardGui") and yb.Enabled then
                    return plot
                end
                local surfaceGui = sign:FindFirstChildWhichIsA("SurfaceGui", true)
                if surfaceGui then
                    local label = surfaceGui:FindFirstChildWhichIsA("TextLabel", true)
                    if label then
                        local text = label.Text:lower()
                        if text:find(LocalPlayer.DisplayName:lower(), 1, true)
                            or text:find(LocalPlayer.Name:lower(), 1, true) then
                            return plot
                        end
                    end
                end
            end
        end
        return nil
    end

    local function getPlotAnchorPart(plot)
        if not plot then return nil end
        local sign = plot:FindFirstChild("PlotSign")
        if sign then
            if sign:IsA("BasePart") then return sign end
            local part = sign:FindFirstChildWhichIsA("BasePart", true)
            if part then return part end
        end
        local main = plot:FindFirstChild("MainRootPart")
        if main and main:IsA("BasePart") then return main end
        return plot:FindFirstChildWhichIsA("BasePart")
    end

    local function ensurePlotBeam(hrp, plotPart)
        if not hrp or not hrp.Parent or not plotPart or not plotPart.Parent then return end

        if not plotAtt0 or not plotAtt0.Parent or plotAtt0.Parent ~= hrp then
            if plotAtt0 then pcall(function() plotAtt0:Destroy() end) end
            plotAtt0 = hrp:FindFirstChild(ATT0_NAME) or Instance.new("Attachment")
            plotAtt0.Name = ATT0_NAME
            plotAtt0.Position = Vector3.new(0, 0, 0)
            plotAtt0.Parent = hrp
        end

        if currentPlotPart ~= plotPart or not plotAtt1 or not plotAtt1.Parent then
            if plotAtt1 then pcall(function() plotAtt1:Destroy() end) end
            plotAtt1 = plotPart:FindFirstChild(ATT1_NAME) or Instance.new("Attachment")
            plotAtt1.Name = ATT1_NAME
            plotAtt1.Position = Vector3.new(0, 4, 0)
            plotAtt1.Parent = plotPart
            currentPlotPart = plotPart
            if plotBeam then plotBeam.Attachment1 = plotAtt1 end
        end

        if not plotBeam or not plotBeam.Parent then
            if plotBeam then pcall(function() plotBeam:Destroy() end) end
            local oldGlow = hrp:FindFirstChild("PlotBeamGlow")
            if oldGlow then pcall(function() oldGlow:Destroy() end) end
            plotBeam = Instance.new("Beam")
            plotBeam.Name = BEAM_NAME
            plotBeam.Attachment0 = plotAtt0
            plotBeam.Attachment1 = plotAtt1
            applyPlotBeamStyle(plotBeam)
            plotBeam.Parent = hrp
        else
            plotBeam.Attachment0 = plotAtt0
            plotBeam.Attachment1 = plotAtt1
            applyPlotBeamStyle(plotBeam)
        end
    end

    local function createPlotBeam()
        if not Config.LineToBase then return end
        local char = LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        local myPlot = findMyPlot()
        if not myPlot then return end
        local plotPart = getPlotAnchorPart(myPlot)
        if not plotPart then return end
        pcall(ensurePlotBeam, hrp, plotPart)
    end

    task.spawn(function()
        local check = 0
        RunService.Heartbeat:Connect(function()
            if not Config.LineToBase then
                if plotBeam or plotAtt0 or plotAtt1 then destroyPlotBeam() end
                return
            end
            check = check + 1
            if check < 10 then return end
            check = 0

            local char = LocalPlayer.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if not hrp then
                if plotBeam or plotAtt0 or plotAtt1 then destroyPlotBeam() end
                return
            end

            local myPlot = findMyPlot()
            if not myPlot then
                if plotBeam or plotAtt1 then
                    if plotBeam then pcall(function() plotBeam:Destroy() end); plotBeam = nil end
                    if plotAtt1 then pcall(function() plotAtt1:Destroy() end); plotAtt1 = nil end
                    currentPlotPart = nil
                end
                return
            end

            local plotPart = getPlotAnchorPart(myPlot)
            if not plotPart or not plotPart.Parent then return end
            pcall(ensurePlotBeam, hrp, plotPart)
        end)
    end)

    LocalPlayer.CharacterAdded:Connect(function()
        task.wait(0.5)
        currentPlotPart = nil
        plotAtt0 = nil
        if plotBeam then pcall(function() plotBeam:Destroy() end); plotBeam = nil end
        if Config.LineToBase then pcall(createPlotBeam) end
    end)

    if LocalPlayer.Character and Config.LineToBase then
        task.defer(createPlotBeam)
    end

    _G.createPlotBeam = createPlotBeam
    _G.resetPlotBeam = destroyPlotBeam
end

-- Line To Best Brainrot (kinqs beam style)
do
    local BEAM_NAME    = "BestPetBeam"
    local ATT0_NAME    = "BestPetBeamAttach_Player"
    local ATT1_NAME    = "BestPetBeamAttach_Target"
    local BEAM_COLOR   = Color3.fromRGB(255, 45, 190)   -- pink (line to best brainrot)

    local bestBeam     = nil
    local bestAtt0     = nil
    local bestAtt1     = nil
    local currentTargetPart = nil

    local function destroyBeam()
        if bestBeam then pcall(function() bestBeam:Destroy() end) end
        if bestAtt0 then pcall(function() bestAtt0:Destroy() end) end
        if bestAtt1 then pcall(function() bestAtt1:Destroy() end) end
        bestBeam, bestAtt0, bestAtt1 = nil, nil, nil
        currentTargetPart = nil
    end

    local function getBestTargetPart()
        local cache = SharedState.AllAnimalsCache
        if not cache or #cache == 0 then return nil end
        local bestData = nil
        local bestVal = 0
        -- Priority list first
        for _, pName in ipairs(priorityList) do
            local searchName = pName:lower()
            for _, a in ipairs(cache) do
                if a and a.name and a.name:lower() == searchName and a.owner ~= LocalPlayer.Name then
                    local score = (a.genValue or 0) + 1e15
                    if score > bestVal then
                        bestVal = score
                        bestData = a
                    end
                end
            end
            if bestData then break end
        end
        -- Then brainrots by value
        if not bestData then
            for _, a in ipairs(cache) do
                if a and a.owner ~= LocalPlayer.Name and a.genValue and a.genValue >= 10000000 then
                    if a.genValue > bestVal then
                        bestVal = a.genValue
                        bestData = a
                    end
                end
            end
        end
        -- Then highest gen fallback
        if not bestData then
            for _, a in ipairs(cache) do
                if a and a.owner ~= LocalPlayer.Name and (a.genValue or 0) >= 1 then
                    if (a.genValue or 0) > bestVal then
                        bestVal = a.genValue or 0
                        bestData = a
                    end
                end
            end
        end
        if not bestData then return nil end
        local adornee = findAdorneeGlobal(bestData)
        if adornee and adornee:IsA("BasePart") then return adornee end
        return nil
    end

    local function ensureBeam(hrp, targetPart)
        if not hrp or not hrp.Parent or not targetPart or not targetPart.Parent then return end
        if not bestAtt0 or not bestAtt0.Parent or bestAtt0.Parent ~= hrp then
            if bestAtt0 then pcall(function() bestAtt0:Destroy() end) end
            bestAtt0 = hrp:FindFirstChild(ATT0_NAME) or Instance.new("Attachment")
            bestAtt0.Name = ATT0_NAME
            bestAtt0.Position = Vector3.new(0, 0, 0)
            bestAtt0.Parent = hrp
        end
        if currentTargetPart ~= targetPart or not bestAtt1 or not bestAtt1.Parent then
            if bestAtt1 then pcall(function() bestAtt1:Destroy() end) end
            bestAtt1 = targetPart:FindFirstChild(ATT1_NAME) or Instance.new("Attachment")
            bestAtt1.Name = ATT1_NAME
            bestAtt1.Position = Vector3.new(0, 3, 0)
            bestAtt1.Parent = targetPart
            currentTargetPart = targetPart
            if bestBeam then
                bestBeam.Attachment1 = bestAtt1
            end
        end
        if not bestBeam or not bestBeam.Parent then
            if bestBeam then pcall(function() bestBeam:Destroy() end) end
            bestBeam = Instance.new("Beam")
            bestBeam.Name = BEAM_NAME
            bestBeam.Attachment0 = bestAtt0
            bestBeam.Attachment1 = bestAtt1
            bestBeam.FaceCamera = true
            bestBeam.LightEmission = 1
            bestBeam.Color = ColorSequence.new(BEAM_COLOR)
            bestBeam.Transparency = NumberSequence.new(0.35)
            bestBeam.Width0 = 0.45
            bestBeam.Width1 = 0.45
            bestBeam.TextureMode = Enum.TextureMode.Wrap
            bestBeam.TextureSpeed = 0
            bestBeam.Parent = hrp
        end
    end

    task.spawn(function()
        local check = 0
        RunService.Heartbeat:Connect(function()
            if not Config.LineToBrainrot then
                if bestBeam or bestAtt0 or bestAtt1 then destroyBeam() end
                return
            end
            check = check + 1
            if check < 10 then return end
            check = 0

            local char = LocalPlayer.Character
            local hrp  = char and char:FindFirstChild("HumanoidRootPart")
            if not hrp then
                if bestBeam or bestAtt0 or bestAtt1 then destroyBeam() end
                return
            end

            local targetPart = getBestTargetPart()
            if not targetPart or not targetPart.Parent then
                if bestBeam or bestAtt1 then
                    if bestBeam then pcall(function() bestBeam:Destroy() end); bestBeam = nil end
                    if bestAtt1 then pcall(function() bestAtt1:Destroy() end); bestAtt1 = nil end
                    currentTargetPart = nil
                end
                return
            end

            pcall(ensureBeam, hrp, targetPart)

            -- Color coding: green when stealing, yellow when close, red default
            if bestBeam then
                local col = BEAM_COLOR
                if LocalPlayer:GetAttribute("Stealing") then
                    col = Color3.fromRGB(80, 255, 120)
                else
                    local dist = hrp and targetPart and (hrp.Position - targetPart.Position).Magnitude or math.huge
                    if dist < 30 then
                        col = Color3.fromRGB(255, 196, 72)
                    end
                end
                bestBeam.Color = ColorSequence.new(col)
            end
        end)
    end)

    LocalPlayer.CharacterAdded:Connect(function()
        task.wait(0.5)
        currentTargetPart = nil
        bestAtt0 = nil
        if bestBeam then pcall(function() bestBeam:Destroy() end); bestBeam = nil end
    end)

    _G.updateBrainrotBeam = function()
        -- beam auto-updates via heartbeat, nothing to do
    end
    _G.resetBrainrotBeam = destroyBeam
    _G.resetBestPetBeam = destroyBeam
end
-- Subspace Mine ESP
subspaceMineESPEnabled=Config.SubspaceMineESP; local subspaceMineESPData={}
function refreshSubspaceMineESP()
    if not subspaceMineESPEnabled then return end; local tools=Workspace:FindFirstChild("ToolsAdds"); if not tools then return end
    local currentMines={}
    for _,obj in ipairs(tools:GetChildren()) do if obj.Name:match("SubspaceTripmine") and obj:IsA("BasePart") then currentMines[obj]=true
        if not subspaceMineESPData[obj] then
            local ownerName=obj.Name:match("SubspaceTripmine(.+)") or "Unknown"
            local sel=Instance.new("SelectionBox",obj); sel.Color3=Color3.fromRGB(167,142,255); sel.LineThickness=0.05
            local bb=Instance.new("BillboardGui",obj); bb.Size=UDim2.new(0,250,0,50); bb.StudsOffset=Vector3.new(0,2.5,0); bb.AlwaysOnTop=false
            local lbl=Instance.new("TextLabel",bb); lbl.Size=UDim2.new(1,0,1,0); lbl.BackgroundTransparency=1; lbl.Text=ownerName.."'s Subspace Mine"
            lbl.TextColor3=Color3.fromRGB(167,142,255); lbl.TextStrokeTransparency=0; lbl.Font=Enum.Font.GothamBold; lbl.TextSize=16
            subspaceMineESPData[obj]={sel=sel,bb=bb}
        end
    end end
    for mineObj,data in pairs(subspaceMineESPData) do if not currentMines[mineObj] or not mineObj.Parent then
        data.sel:Destroy(); data.bb:Destroy(); subspaceMineESPData[mineObj]=nil
    end end
end
task.spawn(function() while true do if subspaceMineESPEnabled then pcall(refreshSubspaceMineESP) end; task.wait(0.5) end end)

-- Timer ESP
timerESPEnabled=Config.TimerESP
function clearTimerESP()
    local plots = Workspace:FindFirstChild("Plots")
    if plots then
        for _, plot in ipairs(plots:GetChildren()) do
            for _, desc in ipairs(plot:GetDescendants()) do
                if desc.Name == "TimerESP" and desc:IsA("BillboardGui") then
                    pcall(function() desc:Destroy() end)
                end
            end
        end
    end
end
_G.clearTimerESP = clearTimerESP

function refreshTimerESP()
    if not timerESPEnabled then
        clearTimerESP()
        return
    end
    local plots=Workspace:FindFirstChild("Plots"); if not plots then return end
    for _,plot in ipairs(plots:GetChildren()) do
        -- The game shows a timer per floor. User wants ONLY the first floor, so
        -- gather this plot's timer billboards and keep just the lowest-Y ones.
        local found = {}
        local minY = math.huge
        for _,g in ipairs(plot:GetDescendants()) do
            if g:IsA("BillboardGui") and g:FindFirstChild("RemainingTime") then
                local base=g.Adornee or g.Parent
                if base and base:IsA("BasePart") then
                    found[#found+1] = {g=g, base=base, y=base.Position.Y}
                    if base.Position.Y < minY then minY = base.Position.Y end
                end
            end
        end
        for _,item in ipairs(found) do
            local base, g = item.base, item.g
            local existing = base:FindFirstChild("TimerESP")
            if item.y <= minY + 4 then   -- first floor only
                local rt = g:FindFirstChild("RemainingTime")
                if rt then
                    if not existing then
                        local bb=Instance.new("BillboardGui"); bb.Name="TimerESP"; bb.Adornee=base; bb.Size=UDim2.new(0,98,0,26); bb.AlwaysOnTop=true; bb.StudsOffsetWorldSpace=Vector3.new(0,1.6,0)
                        local lbl=Instance.new("TextLabel",bb); lbl.Size=UDim2.new(1,0,1,0); lbl.BackgroundTransparency=1; lbl.Text=rt.Text
                        lbl.Font=Enum.Font.GothamBold; lbl.TextSize=15; lbl.TextColor3=Color3.fromRGB(255,255,255)
                        lbl.TextStrokeTransparency=0.35; lbl.TextStrokeColor3=Color3.fromRGB(0,0,0); bb.Parent=base
                    else existing.TextLabel.Text=rt.Text end
                end
            elseif existing then
                pcall(function() existing:Destroy() end)   -- not first floor -> remove
            end
        end
    end
end
task.spawn(function() while true do if timerESPEnabled then pcall(refreshTimerESP) else clearTimerESP() end; task.wait(0.5) end end)

-- ============================================================
-- Base Owner ESP  (red outline + "BASE OWNER" tag on the owner of the
-- base you are CURRENTLY standing in -- one player at a time, not all bases)
-- Wrapped in a do-block + exposed via _G so it adds no chunk-level locals.
-- ============================================================
do
    local enabled = Config.BaseOwnerESP
    local entries = {}   -- userId -> { hl = Highlight, bb = BillboardGui }

    local function destroyEntry(e)
        if not e then return end
        if e.hl then pcall(function() e.hl:Destroy() end) end
        if e.bb then pcall(function() e.bb:Destroy() end) end
    end
    local function clearAll()
        for uid, e in pairs(entries) do destroyEntry(e); entries[uid] = nil end
    end

    local function makeTag()
        local bb = Instance.new("BillboardGui")
        bb.Name = "BaseOwnerTag"
        bb.Size = UDim2.new(0, 160, 0, 34)
        bb.StudsOffsetWorldSpace = Vector3.new(0, 3.6, 0)
        bb.AlwaysOnTop = true
        bb.LightInfluence = 0
        local lbl = Instance.new("TextLabel", bb)
        lbl.Size = UDim2.fromScale(1, 1)
        lbl.BackgroundTransparency = 1
        lbl.Font = Enum.Font.GothamBlack
        lbl.TextSize = 18
        lbl.TextColor3 = Color3.fromRGB(255, 60, 60)
        lbl.TextStrokeTransparency = 0
        lbl.TextStrokeColor3 = Color3.fromRGB(0, 0, 0)
        lbl.Text = "BASE OWNER"
        return bb
    end

    -- Highlight + tag ONLY the given player (owner of the base you're in).
    -- Passing nil clears everything.
    local function setTarget(plr)
        for uid, e in pairs(entries) do
            if not plr or uid ~= plr.UserId then destroyEntry(e); entries[uid] = nil end
        end
        if not plr or not plr.Character then return end
        local uid = plr.UserId
        local e = entries[uid]; if not e then e = {}; entries[uid] = e end
        if not e.hl or not e.hl.Parent then
            if e.hl then pcall(function() e.hl:Destroy() end) end
            local hl = Instance.new("Highlight")
            hl.Name = "BaseOwnerESP"
            hl.FillColor = Color3.fromRGB(255, 0, 0); hl.FillTransparency = 0.6
            hl.OutlineColor = Color3.fromRGB(255, 0, 0); hl.OutlineTransparency = 0
            hl.DepthMode = Enum.HighlightDepthMode.AlwaysOnTop
            hl.Parent = CoreGui
            e.hl = hl
        end
        if e.hl.Adornee ~= plr.Character then e.hl.Adornee = plr.Character end
        if not e.bb or not e.bb.Parent then
            if e.bb then pcall(function() e.bb:Destroy() end) end
            e.bb = makeTag(); e.bb.Parent = CoreGui
        end
        local head = plr.Character:FindFirstChild("Head") or plr.Character:FindFirstChild("HumanoidRootPart")
        if head and e.bb.Adornee ~= head then e.bb.Adornee = head end
    end

    local function refresh()
        local char = LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then setTarget(nil); return end
        local plot = getPlotAtPosition(hrp.Position)
        local owner = plot and getPlotOwner(plot)
        if owner == LocalPlayer then owner = nil end   -- don't tag your own base
        setTarget(owner)
    end
    _G.setBaseOwnerESP = function(on)
        enabled = on
        Config.BaseOwnerESP = on
        saveConfig()
        setToggle("Base Owner ESP", on)
        if on then pcall(refresh) else clearAll() end
    end
    _G.clearBaseOwnerESP = clearAll
    task.spawn(function()
        while true do
            task.wait(0.5)
            if enabled then pcall(refresh) end
        end
    end)
end

-- ============================================================
-- ============================================================
-- FPS BOOST
-- ============================================================
(function()
laserOriginalState = setmetatable({}, {__mode = "k"})
_plotLaserOpenState = {}
_plotMonitorSetup = {}
_plotsFolderHooked = false
_BLOCK_TIME_KEYS = {
    "BlockEndTimeFirstFloor",
    "BlockEndTimeSecondFloor",
    "BlockEndTime",
}

function IsLaserObject(obj)
    if not obj then return false end
    if obj.Name == "Laser" or obj.Name == "LaserHitbox" then return true end
    local name = obj.Name:lower()
    if name:find("laser") then return true end
    local parent = obj.Parent
    while parent and parent ~= Workspace do
        if parent.Name == "Laser" then return true end
        if parent.Name:lower():find("laser") then return true end
        parent = parent.Parent
    end
    return false
end

function GetPlotFromObject(obj)
    local p = obj and obj.Parent
    while p and p ~= Workspace do
        if p.Parent and p.Parent.Name == "Plots" then
            return p
        end
        p = p.Parent
    end
    return nil
end

function readPlotChannel(plotName)
    local Sync = _G.__getSync and _G.__getSync()
    if Sync then
        local ch = nil
        pcall(function() ch = _G.XenSyncGet(plotName) end)
        if not ch then local _t0=os.clock() repeat ch=_G.XenSyncGet(plotName) if ch then break end task.wait(0.05) until os.clock()-_t0>2 end
        if ch then return ch end
    end
    if _G.stealthGet then
        return _G.stealthGet(plotName)
    end
    return nil
end

function readChannelProp(ch, prop)
    if not ch then return nil end
    local v = _G.sProp and _G.sProp(ch, prop)
    if v ~= nil then return v end
    pcall(function()
        if type(ch.Get) == "function" then
            v = ch:Get(prop)
        end
    end)
    if v ~= nil then return v end
    pcall(function()
        if ch.CacheTable then v = ch.CacheTable[prop] end
    end)
    return v
end

function isBlockTimeActive(val)
    if val == nil or val == false then return false end
    if type(val) == "number" then
        return val > os.time()
    end
    -- purchased/cleared floors can leave a table with the end time inside
    if type(val) == "table" then
        local t = rawget(val, "EndTime") or rawget(val, "endTime") or rawget(val, "Time")
        if type(t) == "number" then return t > os.time() end
    end
    return true
end

-- PER-FLOOR open check: a Robux purchase can open ONE floor while the other
-- stays locked -- the old all-keys check kept every laser visible in that case.
function IsPlotFloorOpen(plot, isUpper)
    if not plot then return false end
    local ok, res = pcall(function()
        local ch = readPlotChannel(plot.Name)
        if not ch then return false end
        local key = isUpper and "BlockEndTimeSecondFloor" or "BlockEndTimeFirstFloor"
        if isBlockTimeActive(readChannelProp(ch, key)) then return false end
        if isBlockTimeActive(readChannelProp(ch, "BlockEndTime")) then return false end
        return true
    end)
    return ok and res == true
end

function IsPlotBaseOpen(plot)
    -- fully open = both floors open (kept for the coarse state-change monitor)
    return IsPlotFloorOpen(plot, false) and IsPlotFloorOpen(plot, true)
end

-- which floor does a laser object belong to? (upper floor sits at Y ~17)
function _laserIsUpper(obj)
    local p = obj
    while p and not p:IsA("BasePart") do p = p.Parent end
    if p and p:IsA("BasePart") then return p.Position.Y > 10 end
    return false
end

function SaveLaserOriginal(obj)
    if laserOriginalState[obj] then return end
    pcall(function()
        if obj:IsA("BasePart") then
            local trans = obj.Transparency
            local localTrans = obj.LocalTransparencyModifier
            if trans >= 1 and localTrans >= 1 then
                trans, localTrans = 0, 0
            end
            laserOriginalState[obj] = {
                trans = trans,
                localTrans = localTrans,
                shadow = obj.CastShadow,
            }
        elseif obj:IsA("Beam") or obj:IsA("ParticleEmitter") or obj:IsA("Trail")
            or obj:IsA("Smoke") or obj:IsA("Fire") or obj:IsA("Sparkles") then
            laserOriginalState[obj] = { enabled = obj.Enabled ~= false }
        elseif obj:IsA("Decal") or obj:IsA("Texture") then
            local trans = obj.Transparency
            if trans >= 1 then trans = 0 end
            laserOriginalState[obj] = { trans = trans }
        elseif obj:IsA("GuiObject") then
            laserOriginalState[obj] = { visible = obj.Visible ~= false }
        end
    end)
end

function HideLaserObject(obj)
    if not obj then return end
    pcall(function()
        if obj:IsA("BasePart") then
            obj.Transparency = 1
            obj.LocalTransparencyModifier = 1
            obj.CastShadow = false
        elseif obj:IsA("Beam") or obj:IsA("ParticleEmitter") or obj:IsA("Trail")
            or obj:IsA("Smoke") or obj:IsA("Fire") or obj:IsA("Sparkles") then
            obj.Enabled = false
        elseif obj:IsA("Decal") or obj:IsA("Texture") then
            obj.Transparency = 1
        elseif obj:IsA("GuiObject") then
            obj.Visible = false
        end
    end)
end

function RestoreLaserObject(obj)
    local orig = laserOriginalState[obj]
    pcall(function()
        if obj:IsA("BasePart") then
            local trans = orig and orig.trans or 0
            local localTrans = orig and orig.localTrans or 0
            if trans >= 1 then trans = 0 end
            if localTrans >= 1 then localTrans = 0 end
            obj.Transparency = trans
            obj.LocalTransparencyModifier = localTrans
            obj.CastShadow = orig and orig.shadow or true
        elseif obj:IsA("Beam") or obj:IsA("ParticleEmitter") or obj:IsA("Trail")
            or obj:IsA("Smoke") or obj:IsA("Fire") or obj:IsA("Sparkles") then
            obj.Enabled = orig and orig.enabled or true
        elseif obj:IsA("Decal") or obj:IsA("Texture") then
            local trans = orig and orig.trans or 0
            if trans >= 1 then trans = 0 end
            obj.Transparency = trans
        elseif obj:IsA("GuiObject") then
            obj.Visible = orig and orig.visible or true
        end
    end)
end

function UpdateLaserObject(obj)
    if not IsLaserObject(obj) then return end
    local plot = GetPlotFromObject(obj)
    -- per-floor: hide this laser if ITS floor is open (Robux unlock included)
    if plot and IsPlotFloorOpen(plot, _laserIsUpper(obj)) then
        SaveLaserOriginal(obj)
        HideLaserObject(obj)
    else
        RestoreLaserObject(obj)
    end
end

function RefreshPlotLasers(plot)
    if not plot then return end
    local laserFolder = plot:FindFirstChild("Laser")
    if not laserFolder then return end
    UpdateLaserObject(laserFolder)
    for _, child in ipairs(laserFolder:GetDescendants()) do
        UpdateLaserObject(child)
    end
end

function BindLaserState(obj, plot, connList)
    UpdateLaserObject(obj)
    if obj:IsA("BasePart") then
        connList[#connList + 1] = obj:GetPropertyChangedSignal("Transparency"):Connect(function()
            if not plot then return end
            if IsPlotFloorOpen(plot, _laserIsUpper(obj)) and obj.Transparency < 1 then
                SaveLaserOriginal(obj)
                HideLaserObject(obj)
            elseif not IsPlotFloorOpen(plot, _laserIsUpper(obj)) then
                RestoreLaserObject(obj)
            end
        end)
    elseif obj:IsA("Beam") or obj:IsA("ParticleEmitter") or obj:IsA("Trail")
        or obj:IsA("Smoke") or obj:IsA("Fire") or obj:IsA("Sparkles") then
        connList[#connList + 1] = obj:GetPropertyChangedSignal("Enabled"):Connect(function()
            if not plot then return end
            if IsPlotFloorOpen(plot, _laserIsUpper(obj)) and obj.Enabled then
                SaveLaserOriginal(obj)
                HideLaserObject(obj)
            elseif not IsPlotFloorOpen(plot, _laserIsUpper(obj)) and not obj.Enabled then
                RestoreLaserObject(obj)
            end
        end)
    end
end

function SetupPlotLaserMonitor(plot, connList)
    if not plot then return end
    local laserFolder = plot:FindFirstChild("Laser")
    if not laserFolder then return end
    if _plotMonitorSetup[plot] then return end
    _plotMonitorSetup[plot] = true

    _plotLaserOpenState[plot] = tostring(IsPlotFloorOpen(plot, false)) .. tostring(IsPlotFloorOpen(plot, true))
    RefreshPlotLasers(plot)

    BindLaserState(laserFolder, plot, connList)
    for _, child in ipairs(laserFolder:GetDescendants()) do
        BindLaserState(child, plot, connList)
    end

    connList[#connList + 1] = laserFolder.DescendantAdded:Connect(function(obj)
        task.defer(function()
            BindLaserState(obj, plot, connList)
        end)
    end)

    local frame = 0
    connList[#connList + 1] = RunService.Heartbeat:Connect(function()
        frame = frame + 1
        if frame % 6 ~= 0 then return end
        -- per-floor state string so a SINGLE floor opening (Robux unlock)
        -- triggers the refresh too
        local state = tostring(IsPlotFloorOpen(plot, false)) .. tostring(IsPlotFloorOpen(plot, true))
        if state ~= _plotLaserOpenState[plot] then
            _plotLaserOpenState[plot] = state
            RefreshPlotLasers(plot)
        end
    end)
end

function RefreshAllPlotLasers()
    local plotsFolder = Workspace:FindFirstChild("Plots")
    if not plotsFolder then return end
    for _, plot in ipairs(plotsFolder:GetChildren()) do
        RefreshPlotLasers(plot)
    end
end

function SetupPlotLaserWatch(plot, connList)
    if not plot or _plotMonitorSetup[plot] then return end
    if plot:FindFirstChild("Laser") then
        SetupPlotLaserMonitor(plot, connList)
        return
    end
    connList[#connList + 1] = plot.ChildAdded:Connect(function(child)
        if child.Name == "Laser" then
            SetupPlotLaserMonitor(plot, connList)
        end
    end)
end

function SetupAllPlotLaserMonitor(connList)
    local plotsFolder = Workspace:FindFirstChild("Plots")
    if not plotsFolder then return end
    for _, plot in ipairs(plotsFolder:GetChildren()) do
        SetupPlotLaserWatch(plot, connList)
    end
    if not _plotsFolderHooked then
        _plotsFolderHooked = true
        connList[#connList + 1] = plotsFolder.ChildAdded:Connect(function(plot)
            task.wait(0.2)
            SetupPlotLaserWatch(plot, connList)
        end)
    end
end

_globalLaserConnections = {}
task.defer(function()
    local plotsFolder = Workspace:WaitForChild("Plots", 30)
    if plotsFolder then
        SetupAllPlotLaserMonitor(_globalLaserConnections)
    end
end)

_G.__laser = {
    IsLaserObject = IsLaserObject,
    UpdateLaserObject = UpdateLaserObject,
    SetupPlotLaserMonitor = SetupPlotLaserMonitor,
    SetupAllPlotLaserMonitor = SetupAllPlotLaserMonitor,
    RefreshPlotLasers = RefreshPlotLasers,
    RefreshAllPlotLasers = RefreshAllPlotLasers,
    GetPlotFromObject = GetPlotFromObject,
    IsPlotBaseOpen = IsPlotBaseOpen,
    SaveLaserOriginal = SaveLaserOriginal,
    HideLaserObject = HideLaserObject,
    RestoreLaserObject = RestoreLaserObject,
}
end)()

function IsProtected(obj)
    if not obj then return false end
    if _G.__laser and _G.__laser.IsLaserObject and _G.__laser.IsLaserObject(obj) then return true end
    local name = obj.Name
    if name:find("DO_NOT_REMOVE") or name:find("ForgeTextureCache") or name == "ChargeSoundPart"
        or name == "XiProxAPRing" or name == "MynxxVizPath" then
        return true
    end
    name = name:lower()
    -- "tutorial" protects Workspace.TutorialArrow (FTUE): the Ultra sweep was
    -- destroying its "Start" child -> the game's FTUEController spammed
    -- 'Start is not a valid member of Model "Workspace.TutorialArrow"' forever.
    if name:find("door") or name:find("gate") or name:find("shield") or name:find("barrier") or name:find("fence") or name:find("forcefield") or name:find("wall") or name:find("protect") or name:find("tutorial") then
        return true
    end
    local parent = obj.Parent
    while parent and parent ~= Workspace do
        local pName = parent.Name
        if pName == "CraftingMachine" or pName:find("Forge") or pName:find("Crafting") then return true end
        pName = pName:lower()
        if pName:find("door") or pName:find("gate") or pName:find("shield") or pName:find("barrier") or pName:find("fence") or pName:find("forcefield") or pName:find("wall") or pName:find("protect") or pName:find("tutorial") then
            return true
        end
        parent = parent.Parent
    end
    return false
end

fpsBoostConnection = nil
_fpsBoostLaserConnections = {}
function setFPSBoost(enabled) Config.FPSBoost=enabled; saveConfig()
    setToggle("FPS Boost (normal)", enabled)
    if fpsBoostConnection then
        pcall(function() fpsBoostConnection:Disconnect() end)
        fpsBoostConnection = nil
    end
    for _, conn in ipairs(_fpsBoostLaserConnections) do
        pcall(function() conn:Disconnect() end)
    end
    _fpsBoostLaserConnections = {}
    if enabled then
        Lighting.GlobalShadows=false; Lighting.Brightness=2; Lighting.FogEnd=9e9; Lighting.FogStart=0
        Lighting.EnvironmentDiffuseScale=0; Lighting.EnvironmentSpecularScale=0
        pcall(function() Lighting.ShadowSoftness=0 end)
        for _,v in pairs(Lighting:GetChildren()) do
            if v.Name ~= "MynxxDarkTheme" and (v:IsA("BloomEffect") or v:IsA("BlurEffect") or v:IsA("ColorCorrectionEffect") or v:IsA("SunRaysEffect") or v:IsA("DepthOfFieldEffect")) then pcall(function() v.Enabled=false end) elseif v:IsA("Atmosphere") then pcall(function() v:Destroy() end) end
        end
        local _fpsDescs = Workspace:GetDescendants()
        for i = 1, #_fpsDescs do
            local obj = _fpsDescs[i]
            if _G.__laser.IsLaserObject(obj) then
                _G.__laser.UpdateLaserObject(obj)
            elseif not IsProtected(obj) then
                if obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam") or obj:IsA("Smoke") or obj:IsA("Fire") or obj:IsA("Sparkles") then
                    if not (obj:IsA("Beam") and (obj.Name == "PlotBeam" or obj.Name == "PlotBeamGlow" or obj.Name == "BestPetBeam")) then
                        pcall(function() obj.Enabled=false end)
                    end
                end
                if obj:IsA("BasePart") then pcall(function() obj.Material=Enum.Material.Plastic; obj.CastShadow=false end) end
                if obj:IsA("SurfaceAppearance") or obj:IsA("Texture") or obj:IsA("Decal") then pcall(function() obj:Destroy() end) end
                if obj:IsA("PointLight") or obj:IsA("SpotLight") or obj:IsA("SurfaceLight") then pcall(function() obj.Enabled=false; obj.Shadows=false end) end
            end
            if i % 100 == 0 then task.wait() end
        end
        fpsBoostConnection = Workspace.DescendantAdded:Connect(function(obj)
            if not Config.FPSBoost then return end
            if _G.__laser.IsLaserObject(obj) then
                _G.__laser.UpdateLaserObject(obj)
            elseif not IsProtected(obj) then
                if obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam") or obj:IsA("Smoke") then
                    if not (obj:IsA("Beam") and (obj.Name == "PlotBeam" or obj.Name == "PlotBeamGlow" or obj.Name == "BestPetBeam")) then
                        pcall(function() obj.Enabled=false end)
                    end
                end
                if obj:IsA("BasePart") then pcall(function() obj.Material=Enum.Material.Plastic; obj.CastShadow=false end) end
                if obj:IsA("PointLight") or obj:IsA("SpotLight") or obj:IsA("SurfaceLight") then pcall(function() obj.Enabled=false; obj.Shadows=false end) end
            end
        end)
    end
end

-- FPS BOOST ULTRA
OriginalTransparency = setmetatable({}, {__mode = "k"})
_ultraDescendantConn = nil
_ultraLightingConn = nil
_ultraMaterialConn = nil
_ultraThreads = {}
_ultraConnections = {}

function AddUltraThread(f)
	table.insert(_ultraThreads, task.spawn(f))
end

function AddUltraConnection(c)
	table.insert(_ultraConnections, c)
end

function SafeDestroyUltra(obj)
	if not obj then return end
	if obj.Name == "Overhead" then return end
	if obj.Name:find("DO_NOT_REMOVE") or obj.Name:find("ForgeTextureCache") then return end
	if IsProtected(obj) then return end
	pcall(function() obj:Destroy() end)
end

ClothingClasses = {
	"Shirt","Pants","ShirtGraphic",
	"Accessory","Hat","HairAccessory",
	"FaceAccessory","NeckAccessory","ShoulderAccessory",
	"FrontAccessory","BackAccessory","WaistAccessory",
}

function IsClothing(obj)
	for _, c in ipairs(ClothingClasses) do
		if obj:IsA(c) then return true end
	end
end

function IsCharacterPart(obj)
	local parent = obj.Parent
	while parent and parent ~= Workspace do
		if parent:IsA("Model") and Players:GetPlayerFromCharacter(parent) then
			return true
		end
		parent = parent.Parent
	end
	return false
end

function IsOutOfRange(obj)
	if obj:IsA("BasePart") then
		local x = obj.Position.X
		return x < -560 or x > -240
	end
end

BASE_NAMES = {
	["baseplate"] = true, ["spawnlocation"] = true, ["spawn location"] = true, ["spawn"] = true,
}

function IsBase(obj)
	if not obj:IsA("BasePart") then return false end
	local nameLower = obj.Name:lower()
	if BASE_NAMES[nameLower] then return true end
	for n in pairs(BASE_NAMES) do
		if nameLower:find(n, 1, true) then return true end
	end
	return false
end

function IsInBase(obj)
	local p = obj.Parent
	while p and p ~= workspace do
		if IsBase(p) then return true end
		p = p.Parent
	end
	return false
end

function MakeTransparentUltra(obj)
	pcall(function()
		if IsProtected(obj) then return end
		if IsBase(obj) and not IsCharacterPart(obj) then
			if OriginalTransparency[obj] == nil then
				OriginalTransparency[obj] = {trans = obj.Transparency, shadow = obj.CastShadow}
			end
			obj.Transparency = 1
			obj.CastShadow   = false
		end
	end)
end

function StripObjectUltra(obj)
	pcall(function()
		if IsProtected(obj) then return end
		if obj:IsA("Texture") or obj:IsA("Decal") or obj:IsA("SpecialMesh") then
			SafeDestroyUltra(obj)
		elseif obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam")
			or obj:IsA("Smoke") or obj:IsA("Fire") or obj:IsA("Sparkles") then
			pcall(function() obj.Enabled = false end)
			SafeDestroyUltra(obj)
		elseif obj:IsA("SurfaceAppearance") then
			SafeDestroyUltra(obj)
		elseif obj:IsA("BasePart") then
			obj.CastShadow      = false
			obj.Material        = Enum.Material.Plastic
			obj.MaterialVariant = ""
			obj.Reflectance     = 0
		end
	end)
end

function CleanObjectUltra(obj)
	pcall(function()
		if IsProtected(obj) then return end
		if obj:IsA("SurfaceAppearance") then
			SafeDestroyUltra(obj)
		elseif obj:IsA("Decal") or obj:IsA("Texture") then
			if not (obj.Name == "face" and obj.Parent and obj.Parent.Name == "Head") then
				SafeDestroyUltra(obj)
			end
		elseif obj:IsA("SpecialMesh") then
			obj.TextureId = ""
		elseif obj:IsA("ParticleEmitter") or obj:IsA("Trail") or obj:IsA("Beam") then
			SafeDestroyUltra(obj)
		elseif obj:IsA("PointLight") or obj:IsA("SpotLight") or obj:IsA("SurfaceLight") then
			SafeDestroyUltra(obj)
		elseif obj:IsA("Fire") or obj:IsA("Smoke") or obj:IsA("Sparkles") or obj:IsA("Explosion") then
			SafeDestroyUltra(obj)
		elseif obj:IsA("Animation") or obj:IsA("AnimationController") then
			SafeDestroyUltra(obj)
		elseif obj:IsA("BasePart") then
			obj.CastShadow      = false
			obj.Material        = Enum.Material.Plastic
			obj.MaterialVariant = ""
			obj.Reflectance     = 0
		end
	end)
end

function StopAnimationsUltra(animator)
	pcall(function()
		for _, track in ipairs(animator:GetPlayingAnimationTracks()) do
			local isChar = false
			for _, plr in ipairs(Players:GetPlayers()) do
				if plr.Character and animator:IsDescendantOf(plr.Character) then
					isChar = true
					break
				end
			end
			if not isChar then
				track:Stop()
			end
		end
	end)
end

function OptimizeCharacterUltra(char)
	if not char then return end
	task.spawn(function()
		task.wait(0.3)
		for _, obj in ipairs(char:GetDescendants()) do
			if IsClothing(obj) then
				SafeDestroyUltra(obj)
			else
				CleanObjectUltra(obj)
			end
		end
	end)
end

function ApplyGreySkyUltra()
	pcall(function()
		for _, obj in ipairs(Lighting:GetChildren()) do
			if obj:IsA("Sky") then
				obj:Destroy()
			end
		end
		local sky        = Instance.new("Sky")
		sky.SkyboxBk     = ""
		sky.SkyboxDn     = ""
		sky.SkyboxFt     = ""
		sky.SkyboxLf     = ""
		sky.SkyboxRt     = ""
		sky.SkyboxUp     = ""
		sky.CelestialBodiesShown = false
		sky.StarCount    = 0
		sky.Parent       = Lighting
		-- Force a flat grey sky (blank skybox alone renders dark)
		Lighting.OutdoorAmbient = Color3.fromRGB(128, 128, 128)
		Lighting.ColorShift_Top = Color3.fromRGB(128, 128, 128)
		Lighting.ColorShift_Bottom = Color3.fromRGB(128, 128, 128)
		Lighting.FogColor = Color3.fromRGB(128, 128, 128)
		Lighting.ClockTime = 12
	end)
end

function OptimizeLightingUltra()
	Lighting.GlobalShadows            = false
	Lighting.FogEnd                   = 9e9
	Lighting.FogStart                 = 9e9
	Lighting.EnvironmentDiffuseScale  = 0
	Lighting.EnvironmentSpecularScale = 0
	Lighting.Brightness               = 1.5
	Lighting.Ambient                  = Color3.fromRGB(60, 60, 60)

	for _, v in ipairs(Lighting:GetChildren()) do
		if v:IsA("PostEffect") and v.Name ~= "MynxxDarkTheme" then
			pcall(function() v.Enabled = false end)
		elseif v:IsA("Atmosphere") or v:IsA("Clouds") then
			v:Destroy()
		end
	end
	ApplyGreySkyUltra()
end

function ApplyTerrainUltra()
	pcall(function()
		local T = Workspace.Terrain
		T.Decoration        = false
		T.WaterWaveSize     = 0
		T.WaterWaveSpeed    = 0
		T.WaterReflectance  = 0
		T.WaterTransparency = 1
	end)
end

function setFPSBoostUltra(enabled)
    Config.FPSBoostUltra = enabled
    saveConfig()
    setToggle("FPS Boost Ultra", enabled)
    
    if enabled then
        pcall(function()
            settings().Rendering.QualityLevel        = Enum.QualityLevel.Level01
            settings().Rendering.MeshPartDetailLevel = Enum.MeshPartDetailLevel.Level01
            settings().Physics.AllowSleep = true
        end)
        pcall(function() settings().Physics.PhysicsEnvironmentalThrottle = Enum.EnviromentalPhysicsThrottle.Skip end)
        -- Compatibility lighting = cheapest render pipeline (needs rejoin to fully revert)
        pcall(function()
            if sethiddenproperty then
                sethiddenproperty(Lighting, "Technology", Enum.Technology.Compatibility)
            end
        end)

        OptimizeLightingUltra()
        ApplyTerrainUltra()

        local allDesc = Workspace:GetDescendants()
        local BATCH_SIZE = 200
        for i = 1, #allDesc, BATCH_SIZE do
            local batchEnd = math.min(i + BATCH_SIZE - 1, #allDesc)
            for j = i, batchEnd do
                local obj = allDesc[j]
                if obj and obj.Parent then
                    if IsProtected(obj) then
                        -- skip forge / crafting / locked game assets
                    elseif _G.__laser.IsLaserObject(obj) then
                        _G.__laser.UpdateLaserObject(obj)
                    elseif IsBase(obj) then
                        MakeTransparentUltra(obj)
                    elseif IsClothing(obj) then
                        SafeDestroyUltra(obj)
                    elseif IsInBase(obj) then
                        -- skip
                    elseif IsCharacterPart(obj) then
                        -- skip
                    elseif IsOutOfRange(obj) then
                        SafeDestroyUltra(obj)
                    else
                        CleanObjectUltra(obj)
                        StripObjectUltra(obj)
                        if obj:IsA("Animator") then
                            StopAnimationsUltra(obj)
                        end
                    end
                end
            end
            if i + BATCH_SIZE <= #allDesc then task.wait() end
        end

        AddUltraConnection(Workspace.DescendantAdded:Connect(function(obj)
            task.defer(function()
                if not Config.FPSBoostUltra then return end
                if IsProtected(obj) then return end
                if _G.__laser.IsLaserObject(obj) then
                    _G.__laser.UpdateLaserObject(obj)
                    return
                end
                if IsBase(obj) then
                    MakeTransparentUltra(obj)
                    return
                end
                if IsClothing(obj) then
                    SafeDestroyUltra(obj)
                elseif IsInBase(obj) then
                    -- skip
                elseif IsCharacterPart(obj) then
                    -- skip
                elseif IsOutOfRange(obj) then
                    SafeDestroyUltra(obj)
                else
                    CleanObjectUltra(obj)
                    StripObjectUltra(obj)
                    if obj:IsA("Animator") then
                        StopAnimationsUltra(obj)
                    end
                end
            end)
        end))

        AddUltraConnection(Lighting.DescendantAdded:Connect(function(obj)
            if obj:IsA("PostEffect") and obj.Name ~= "MynxxDarkTheme" then
                pcall(function() obj.Enabled = false end)
            elseif obj:IsA("Atmosphere") or obj:IsA("Clouds") then
                SafeDestroyUltra(obj)
            end
        end))

        local MaterialService = game:GetService("MaterialService")
        for _, obj in ipairs(MaterialService:GetChildren()) do
            SafeDestroyUltra(obj)
        end
        AddUltraConnection(MaterialService.DescendantAdded:Connect(function(obj)
            SafeDestroyUltra(obj)
        end))

        for _, plr in ipairs(Players:GetPlayers()) do
            OptimizeCharacterUltra(plr.Character)
            AddUltraConnection(plr.CharacterAdded:Connect(OptimizeCharacterUltra))
        end

        AddUltraConnection(Players.PlayerAdded:Connect(function(plr)
            AddUltraConnection(plr.CharacterAdded:Connect(OptimizeCharacterUltra))
        end))

        -- GC Loop
        AddUltraThread(function()
            while Config.FPSBoostUltra do
                task.wait(15)
                pcall(function() collectgarbage("collect") end)
            end
        end)
    else
        -- Clean up all connections and threads
        for _, conn in ipairs(_ultraConnections) do
            if typeof(conn) == "RBXScriptConnection" then
                conn:Disconnect()
            end
        end
        _ultraConnections = {}
        for _, thr in ipairs(_ultraThreads) do
            pcall(function() task.cancel(thr) end)
        end
        _ultraThreads = {}

        -- Restore Baseplates opacity
        pcall(function()
            for part, data in pairs(OriginalTransparency) do
                if part and part.Parent then
                    part.Transparency = data.trans
                    part.CastShadow = data.shadow
                end
            end
        end)
        OriginalTransparency = {}

        pcall(function()
            settings().Rendering.QualityLevel = Enum.QualityLevel.Automatic
            settings().Rendering.MeshPartDetailLevel = Enum.MeshPartDetailLevel.Automatic
            Lighting.GlobalShadows = true
            Lighting.Brightness = 2
            Lighting.FogEnd = 100000
            Workspace.Terrain.WaterWaveSize = 0.15
            Workspace.Terrain.WaterWaveSpeed = 1
            Workspace.Terrain.WaterReflectance = 0.5
            Workspace.Terrain.WaterTransparency = 0.3
            Workspace.Terrain.Decoration = true
        end)
    end
end

-- ============================================================
-- KASHI OPTIMIZER
-- ============================================================
do -- KASHI OPTIMIZER SCOPE
    local _kashiConnections = {}
    local _kashiThreads = {}
    local _kashiGraphicsEnabled = true
    local _kashiTexturesRemoved = 0
    local _kashiEffectsDisabled = 0
    local _kashiDarkModeEnabled = false

    local function _kashiGetFFlagSetter()
        local candidates = {"setfflag", "setfastflag", "set_fflag", "setFFlag"}
        for _, name in ipairs(candidates) do
            local fn = rawget(getfenv(), name) or (getgenv and getgenv()[name])
            if type(fn) == "function" then return fn end
        end
        return nil
    end

    local _kashiFFlags = {
        ["DFIntTaskSchedulerTargetFps"] = "9999",
        ["FFlagTaskSchedulerLimitTargetFpsToDevice"] = "False",
        ["FFlagTaskSchedulerLimitTargetFpsTo2402"] = "False",
        ["FIntTaskSchedulerAutoThreadLimit"] = "8",
        ["DFIntTaskSchedulerJobInitThreads"] = "8",
        ["DFIntTaskSchedulerJobInGameThreads"] = "8",
        ["FIntRuntimeMaxNumOfThreads"] = "1000000",
        ["DFIntRuntimeConcurrency"] = "16",
        ["FFlagEnableTextureStreaming"] = "True",
        ["DFFlagEnableSoundPreloading"] = "True",
        ["DFFlagEnableMeshPreloading"] = "True",
        ["DFFlagEnableTexturePreloading"] = "True",
        ["DFIntAssetPreloading"] = "2147483647",
        ["DFFlagEnablePreloadAvatarAssets"] = "True",
        ["DFFlagLuaAppFastGameStartup"] = "True",
        ["FFlagFastLuaAppStartup"] = "True",
        ["FFlagEnableQuickGameLaunch"] = "True",
        ["DFIntMaxConcurrentDownloads"] = "64",
        ["FIntContentProviderConcurrentDownloadCount"] = "64",
        ["FFlagDisablePostProcess"] = "True",
        ["FFlagDisablePostFx"] = "True",
        ["FFlagDisableHQShadersLowEndDx112"] = "True",
        ["FFlagFastGPULightCulling3"] = "True",
        ["FFlagDebugForceFutureIsBrightPhase3"] = "True",
        ["FFlagDebugDisplayFPS"] = "True",
        ["FFlagDisableTelemetryOnJoin"] = "True",
        ["FFlagDebugDisableTelemetryV2Event"] = "True",
        ["FFlagDebugDisableTelemetryV2Stat"] = "True",
        ["FFlagEnableTelemetryProtocol"] = "False",
        ["FFlagLuauCodegen"] = "True",
        ["FFlagLuauCodegenFull"] = "True",
    }

    local function _kashiApplyFFlags()
        local setter = _kashiGetFFlagSetter()
        if not setter then
            warn("[Kashi Optimizer] No FFlag setter found!")
            return
        end
        local applied = 0
        for flagName, flagValue in pairs(_kashiFFlags) do
            pcall(function()
                setter(flagName, flagValue)
                applied += 1
            end)
        end
        print("[Kashi Optimizer] Applied " .. applied .. " FFlags")
    end

    local function _kashiProcessObject(obj)
        if not _kashiGraphicsEnabled then return end
        local class = obj.ClassName

        if class == "ParticleEmitter" or class == "Fire" or class == "Smoke" or class == "Sparkles" or class == "Trail" then
            obj:Destroy()
            _kashiEffectsDisabled += 1
        elseif class == "Decal" or class == "Texture" then
            obj:Destroy()
            _kashiTexturesRemoved += 1
        elseif class == "PointLight" or class == "SpotLight" or class == "SurfaceLight" then
            obj:Destroy()
            _kashiEffectsDisabled += 1
        elseif obj:IsA("BasePart") then
            pcall(function() obj.CastShadow = false end)
        end
    end

    local function _kashiApplyGraphicsOptimizations()
        _kashiTexturesRemoved = 0
        _kashiEffectsDisabled = 0

        for _, obj in ipairs(workspace:GetDescendants()) do
            _kashiProcessObject(obj)
        end

        local Lighting = game:GetService("Lighting")
        Lighting.FogEnd = 100000
        Lighting.FogStart = 99999
        Lighting.GlobalShadows = false
        Lighting.Brightness = 0.5
        Lighting.ClockTime = 12

        for _, obj in ipairs(Lighting:GetChildren()) do
            if obj:IsA("Atmosphere") or obj:IsA("BloomEffect") or obj:IsA("DepthOfFieldEffect") or obj:IsA("SunRays") then
                obj:Destroy()
            end
        end

        settings().Rendering.QualityLevel = Enum.QualityLevel.Level01
        settings().Physics.PhysicsEnvironmentalThrottle = Enum.EnviromentalPhysicsThrottle.Disabled
        settings().Physics.AllowSleep = true

        print("[Kashi Optimizer] Graphics Optimizations Applied")
    end

    local function _kashiApplyDarkMode()
        _kashiDarkModeEnabled = true
        local Lighting = game:GetService("Lighting")
        Lighting.Brightness = 2.2
        Lighting.Ambient = Color3.fromRGB(180, 225, 255)
        Lighting.OutdoorAmbient = Color3.fromRGB(160, 215, 255)
        Lighting.FogColor = Color3.fromRGB(200, 235, 255)
        Lighting.FogEnd = 500
        Lighting.FogStart = 100
        print("[Kashi Optimizer] Ocean Mode ENABLED (Light Blue)")
    end

    local function _kashiDisableDarkMode()
        _kashiDarkModeEnabled = false
        local Lighting = game:GetService("Lighting")
        Lighting.Brightness = 1
        Lighting.Ambient = Color3.fromRGB(70, 70, 70)
        Lighting.OutdoorAmbient = Color3.fromRGB(70, 70, 70)
        Lighting.FogColor = Color3.fromRGB(191, 191, 191)
    end

    local function _kashiCountBrainrot()
        local count = 0
        for _, obj in ipairs(workspace:GetDescendants()) do
            local c = obj.ClassName
            if c == "ParticleEmitter" or c == "Fire" or c == "Smoke" or c == "Sparkles" or c == "Trail" or
               c == "Decal" or c == "Texture" or
               c == "PointLight" or c == "SpotLight" or c == "SurfaceLight" then
                count += 1
            end
        end
        return count
    end

    function setKashiOptimizer(on)
        if on then
            Config.KashiOptimizer = true
            saveConfig()
            _kashiApplyFFlags()
            _kashiApplyGraphicsOptimizations()

            local brainrotLevel = _kashiCountBrainrot()
            if brainrotLevel < 25 then
                _kashiApplyDarkMode()
            end

            -- Listeners for new objects
            table.insert(_kashiConnections, workspace.DescendantAdded:Connect(function(obj)
                task.defer(function()
                    if not Config.KashiOptimizer then return end
                    _kashiProcessObject(obj)
                end)
            end))

            local Lighting = game:GetService("Lighting")
            table.insert(_kashiConnections, Lighting.ChildAdded:Connect(function(obj)
                if _kashiGraphicsEnabled and (obj:IsA("Atmosphere") or obj:IsA("BloomEffect") or obj:IsA("DepthOfFieldEffect")) then
                    obj:Destroy()
                end
            end))

            -- Periodic cleanup
            local thread = task.spawn(function()
                while Config.KashiOptimizer do
                    task.wait(30)
                    if not Config.KashiOptimizer then break end
                    pcall(function() collectgarbage("collect") end)
                end
            end)
            table.insert(_kashiThreads, thread)

            print("[Kashi Optimizer] Fully Loaded | Brainrot: " .. brainrotLevel .. " | Ocean Mode: " .. (_kashiDarkModeEnabled and "ON (Light Blue)" or "OFF"))
        else
            Config.KashiOptimizer = false
            saveConfig()
            _kashiGraphicsEnabled = false
            _kashiDisableDarkMode()

            -- Cleanup connections
            for _, conn in ipairs(_kashiConnections) do
                if typeof(conn) == "RBXScriptConnection" then
                    conn:Disconnect()
                end
            end
            _kashiConnections = {}

            for _, thr in ipairs(_kashiThreads) do
                pcall(function() task.cancel(thr) end)
            end
            _kashiThreads = {}

            -- Restore some settings
            pcall(function()
                settings().Rendering.QualityLevel = Enum.QualityLevel.Automatic
                settings().Physics.PhysicsEnvironmentalThrottle = Enum.EnviromentalPhysicsThrottle.Default
                local Lighting = game:GetService("Lighting")
                Lighting.GlobalShadows = true
                Lighting.Brightness = 2
                Lighting.FogEnd = 100000
            end)

            print("[Kashi Optimizer] Disabled and cleaned up")
        end
    end

    -- Initialize if enabled in config
    if Config.KashiOptimizer then
        task.spawn(function()
            if not game:IsLoaded() then game.Loaded:Wait() end
            setKashiOptimizer(true)
        end)
    end
end

-- ============================================================
-- X-RAY
-- ============================================================
do -- X-RAY SCOPE
xrayOriginalTransparencies = setmetatable({}, {__mode = "k"})
xrayConnections = {}
xrayLoopId = 0

function setXRayTargetTransparency(instance, alphaPercent, loopId)
    if not instance then return end
    if loopId and loopId ~= xrayLoopId then return end   -- X-ray already turned off

    local function apply(obj)
        if _G.__laser.IsLaserObject(obj) then
            local plot = _G.__laser.GetPlotFromObject(obj)
            if plot and _G.__laser.IsPlotBaseOpen(plot) then
                _G.__laser.SaveLaserOriginal(obj)
                _G.__laser.HideLaserObject(obj)
            end
            return
        end
        if obj:IsA("BasePart") then
            if xrayOriginalTransparencies[obj] == nil then
                if obj.Transparency == alphaPercent then xrayOriginalTransparencies[obj] = 0
                else xrayOriginalTransparencies[obj] = obj.Transparency end
            end
            local orig = xrayOriginalTransparencies[obj]
            if orig < 1 then
                local target = orig + (1 - orig) * alphaPercent
                if math.abs(obj.Transparency - target) > 0.01 then obj.Transparency = target end
            end
        elseif obj:IsA("TextLabel") or obj:IsA("TextButton") then
            if xrayOriginalTransparencies[obj] == nil then
                local t, b = obj.TextTransparency, obj.BackgroundTransparency
                if t == alphaPercent then t = 0 end
                if b == alphaPercent then b = 0 end
                xrayOriginalTransparencies[obj] = {text = t, bg = b}
            end
            local orig = xrayOriginalTransparencies[obj]
            if orig.text < 1 then
                local targetText = orig.text + (1 - orig.text) * alphaPercent
                if math.abs(obj.TextTransparency - targetText) > 0.01 then obj.TextTransparency = targetText end
            end
            if orig.bg < 1 then
                local targetBg = orig.bg + (1 - orig.bg) * alphaPercent
                if math.abs(obj.BackgroundTransparency - targetBg) > 0.01 then obj.BackgroundTransparency = targetBg end
            end
        elseif obj:IsA("Frame") or obj:IsA("ScrollingFrame") then
            if xrayOriginalTransparencies[obj] == nil then
                if obj.BackgroundTransparency == alphaPercent then xrayOriginalTransparencies[obj] = 0
                else xrayOriginalTransparencies[obj] = obj.BackgroundTransparency end
            end
            local orig = xrayOriginalTransparencies[obj]
            if orig < 1 then
                local target = orig + (1 - orig) * alphaPercent
                if math.abs(obj.BackgroundTransparency - target) > 0.01 then obj.BackgroundTransparency = target end
            end
        elseif obj:IsA("ImageLabel") or obj:IsA("ImageButton") then
            if xrayOriginalTransparencies[obj] == nil then
                local i, b = obj.ImageTransparency, obj.BackgroundTransparency
                if i == alphaPercent then i = 0 end
                if b == alphaPercent then b = 0 end
                xrayOriginalTransparencies[obj] = {img = i, bg = b}
            end
            local orig = xrayOriginalTransparencies[obj]
            if orig.img < 1 then
                local targetImg = orig.img + (1 - orig.img) * alphaPercent
                if math.abs(obj.ImageTransparency - targetImg) > 0.01 then obj.ImageTransparency = targetImg end
            end
            if orig.bg < 1 then
                local targetBg = orig.bg + (1 - orig.bg) * alphaPercent
                if math.abs(obj.BackgroundTransparency - targetBg) > 0.01 then obj.BackgroundTransparency = targetBg end
            end
        end
    end

    apply(instance)
    local descendants = instance:GetDescendants()
    for i, child in ipairs(descendants) do
        apply(child)
        if i % 300 == 0 then
            task.wait()
            if loopId and loopId ~= xrayLoopId then return end   -- aborted: X-ray turned off mid-apply
        end
    end
end

XRAY_FOLDERS = {"Base","PlotSign","FriendPanel","Cash","Decorations","Skin","Unlock","Purchases"}

-- X-ray a subtree once, then keep newly-streamed parts x-rayed via a cheap
-- DescendantAdded hook. Event-driven -> ZERO cost while nothing changes
-- (replaces the old 1.5s full re-walk that caused the lag).
function trackXRaySubtree(root, alphaPercent, loopId)
    if not root then return end
    if loopId ~= xrayLoopId then return end
    setXRayTargetTransparency(root, alphaPercent, loopId)
    if loopId ~= xrayLoopId then return end   -- don't re-arm a listener after X-ray was turned off
    xrayConnections[#xrayConnections+1] = root.DescendantAdded:Connect(function(obj)
        if loopId ~= xrayLoopId then return end
        setXRayTargetTransparency(obj, alphaPercent, loopId)
    end)
end

function processPlotXRay(plot, alphaPercent, loopId)
    if not plot then return end
    if loopId ~= xrayLoopId then return end

    for _, fname in ipairs(XRAY_FOLDERS) do
        if loopId ~= xrayLoopId then return end
        trackXRaySubtree(plot:FindFirstChild(fname), alphaPercent, loopId)
    end
    if loopId ~= xrayLoopId then return end
    -- Laser visibility is handled by the global _G.__laser monitor.
    if loopId ~= xrayLoopId then return end
    -- target folders that stream in after enable
    xrayConnections[#xrayConnections+1] = plot.ChildAdded:Connect(function(child)
        if loopId ~= xrayLoopId then return end
        for _, fname in ipairs(XRAY_FOLDERS) do
            if child.Name == fname then trackXRaySubtree(child, alphaPercent, loopId); break end
        end
    end)

    local animalPodiums = plot:FindFirstChild("AnimalPodiums")
    if animalPodiums then
        local function processPodium(podium)
            for _, child in ipairs(podium:GetChildren()) do
                if child.Name == "Claim" then
                    trackXRaySubtree(child, alphaPercent, loopId)
                elseif child.Name == "Base" then
                    trackXRaySubtree(child:FindFirstChild("Decorations"), alphaPercent, loopId)
                elseif child:IsA("Model") and child.Name ~= "Decorations" then
                    trackXRaySubtree(child, alphaPercent, loopId)
                end
            end
        end
        for _, podium in ipairs(animalPodiums:GetChildren()) do processPodium(podium) end
        xrayConnections[#xrayConnections+1] = animalPodiums.ChildAdded:Connect(function(podium)
            if loopId ~= xrayLoopId then return end
            task.wait(0.1)
            if loopId ~= xrayLoopId then return end
            processPodium(podium)
        end)
    end
end

function applyTransparencyToAllPlotsXRay(alphaPercent, loopId)
    local plotsFolder = Workspace:FindFirstChild("Plots")
    if not plotsFolder then return end

    for _, plot in ipairs(plotsFolder:GetChildren()) do
        if loopId ~= xrayLoopId then return end
        processPlotXRay(plot, alphaPercent, loopId)
        task.wait()
    end
    -- new bases joining later
    xrayConnections[#xrayConnections+1] = plotsFolder.ChildAdded:Connect(function(plot)
        if loopId ~= xrayLoopId then return end
        task.wait(0.2)
        processPlotXRay(plot, alphaPercent, loopId)
    end)
end

function setXRay(enabled)
    Config.XRay = enabled
    saveConfig()
    setToggle("XRay", enabled)
    setToggle("X-Ray", enabled)
    setToggle("Xray", enabled)

    -- Clean up previous connections
    for _, conn in ipairs(xrayConnections) do
        if typeof(conn) == "RBXScriptConnection" then
            conn:Disconnect()
        end
    end
    xrayConnections = {}

    xrayLoopId = xrayLoopId + 1
    local currentLoopId = xrayLoopId

    if enabled then
        local alphaPercent = 0.5
        
        task.spawn(function()
            -- AÈ™teptÄƒm sÄƒ se Ã®ncarce folder-ul Plots
            while currentLoopId == xrayLoopId and not Workspace:FindFirstChild("Plots") do
                task.wait(0.5)
            end
            
            if currentLoopId ~= xrayLoopId then return end
            
            -- Initial call cu un mic delay sÄƒ dÄƒm timp primelor modele sÄƒ aparÄƒ
            if currentLoopId ~= xrayLoopId then return end
            pcall(applyTransparencyToAllPlotsXRay, alphaPercent, currentLoopId)
            
            -- Loop continuu È™i super-optimizat (fÄƒrÄƒ overlap de thread-uri)
            -- (perpetual 1.5s re-walk removed; upkeep is now event-driven)
        end)
    else
        -- Turn OFF instantly AND completely: swap the snapshot out atomically, then
        -- restore every captured original in ONE synchronous pass (no task.wait
        -- yields). This removes the old race where a restore could overlap a
        -- re-enable and leave some parts still see-through after toggling off.
        local snapshot = xrayOriginalTransparencies
        xrayOriginalTransparencies = setmetatable({}, {__mode = "k"})
        for obj, orig in pairs(snapshot) do
            pcall(function()
                if obj:IsA("BasePart") then
                    obj.Transparency = orig
                elseif obj:IsA("TextLabel") or obj:IsA("TextButton") then
                    obj.TextTransparency = orig.text
                    obj.BackgroundTransparency = orig.bg
                elseif obj:IsA("Frame") or obj:IsA("ScrollingFrame") then
                    obj.BackgroundTransparency = orig
                elseif obj:IsA("ImageLabel") or obj:IsA("ImageButton") then
                    obj.ImageTransparency = orig.img
                    obj.BackgroundTransparency = orig.bg
                end
            end)
        end
        -- User setting: X-ray OFF forces bases solid even when FPS Boost Ultra hid
        -- them. FPS Boost Ultra records each base's true transparency before hiding
        -- it, so re-apply those originals here -> hidden bases become visible again.
        for part, data in pairs(OriginalTransparency) do
            if part and part.Parent and typeof(data) == "table" then
                pcall(function() part.Transparency = data.trans end)
            end
        end
        if _G.__laser and _G.__laser.RefreshAllPlotLasers then
            pcall(_G.__laser.RefreshAllPlotLasers)
        end
    end
end

_G.setXRay = setXRay
end -- END X-RAY SCOPE


-- ============================================================
-- ANTI BEE & DISCO (FIX: FOV desbloqueado + movimento mobile)
-- ============================================================
SharedState.ANTI_BEE_DISCO = {
    running = false,
    connections = {},
    originalMoveFunction = nil,
    controlsProtected = false,
    badLightingNames = { Blue = true, DiscoEffect = true, BeeBlur = true, ColorCorrection = true },
    originalFOV = nil,
    cameraLockConn = nil,
}

SharedState.ANTI_BEE_DISCO.nuke = function(obj)
    if not obj or not obj.Parent then return end
    if SharedState.ANTI_BEE_DISCO.badLightingNames[obj.Name] then
        pcall(function() obj:Destroy() end)
    end
end

SharedState.ANTI_BEE_DISCO.disconnectAll = function()
    for _, conn in ipairs(SharedState.ANTI_BEE_DISCO.connections) do
        if typeof(conn) == "RBXScriptConnection" then conn:Disconnect() end
    end
    SharedState.ANTI_BEE_DISCO.connections = {}
    if SharedState.ANTI_BEE_DISCO.cameraLockConn then
        SharedState.ANTI_BEE_DISCO.cameraLockConn:Disconnect()
        SharedState.ANTI_BEE_DISCO.cameraLockConn = nil
    end
end

-- Protege os controles (com suporte para mobile)
SharedState.ANTI_BEE_DISCO.protectControls = function()
    if SharedState.ANTI_BEE_DISCO.controlsProtected then return end
    pcall(function()
        local PlayerScripts = LocalPlayer.PlayerScripts
        local PlayerModule = PlayerScripts:FindFirstChild("PlayerModule")
        if not PlayerModule then return end
        local Controls = require(PlayerModule):GetControls()
        if not Controls then return end
        local ab = SharedState.ANTI_BEE_DISCO
        if not ab.originalMoveFunction then ab.originalMoveFunction = Controls.moveFunction end
        
        local function protectedMoveFunction(self, moveVector, relativeToCamera)
            -- Se for mobile (touch), não bloqueia o movimento
            if UIS.TouchEnabled then
                if ab.originalMoveFunction then
                    ab.originalMoveFunction(self, moveVector, relativeToCamera)
                end
                return
            end

            -- PC: verifica se alguma tecla de movimento está pressionada
            local isMoving = UIS:IsKeyDown(Enum.KeyCode.W) or UIS:IsKeyDown(Enum.KeyCode.A) or
                             UIS:IsKeyDown(Enum.KeyCode.S) or UIS:IsKeyDown(Enum.KeyCode.D) or
                             UIS:IsKeyDown(Enum.KeyCode.Up) or UIS:IsKeyDown(Enum.KeyCode.Down) or
                             UIS:IsKeyDown(Enum.KeyCode.Left) or UIS:IsKeyDown(Enum.KeyCode.Right)
            local finalVector = isMoving and moveVector or Vector3.zero
            if ab.originalMoveFunction then
                ab.originalMoveFunction(self, finalVector, relativeToCamera)
            end
        end

        table.insert(ab.connections, RunService.Heartbeat:Connect(function()
            if not ab.running or not Config.AntiBeeDisco then return end
            if _G._isTpMoving then return end
            if Controls.moveFunction ~= protectedMoveFunction then
                Controls.moveFunction = protectedMoveFunction
            end
        end))
        Controls.moveFunction = protectedMoveFunction
        ab.controlsProtected = true
    end)
end

SharedState.ANTI_BEE_DISCO.restoreControls = function()
    if not SharedState.ANTI_BEE_DISCO.controlsProtected then return end
    pcall(function()
        local PlayerModule = LocalPlayer.PlayerScripts:FindFirstChild("PlayerModule")
        if not PlayerModule then return end
        local Controls = require(PlayerModule):GetControls()
        local ab = SharedState.ANTI_BEE_DISCO
        if Controls and ab.originalMoveFunction then
            Controls.moveFunction = ab.originalMoveFunction
            ab.controlsProtected = false
        end
    end)
end

-- TRAVA FOV (REMOVER SE NÃO QUISER BLOQUEAR O SLIDER)
-- Se quiser manter a trava, descomente. Se não, mantenha comentado.
SharedState.ANTI_BEE_DISCO.lockCamera = function()
    local ab = SharedState.ANTI_BEE_DISCO
    if ab.cameraLockConn then return end
    local cam = workspace.CurrentCamera
    if not cam then return end
    ab.originalFOV = cam.FieldOfView
    ab.cameraLockConn = RunService.RenderStepped:Connect(function()
        if not ab.running or not Config.AntiBeeDisco then
            return
        end
        pcall(function()
            if cam.FieldOfView ~= (ab.originalFOV or 70) then
                cam.FieldOfView = ab.originalFOV or 70
            end
        end)
    end)
    table.insert(ab.connections, ab.cameraLockConn)
end

SharedState.ANTI_BEE_DISCO.restoreCamera = function()
    local ab = SharedState.ANTI_BEE_DISCO
    if ab.cameraLockConn then
        ab.cameraLockConn:Disconnect()
        ab.cameraLockConn = nil
    end
    local cam = workspace.CurrentCamera
    if cam and ab.originalFOV then
        cam.FieldOfView = ab.originalFOV
    end
end

SharedState.ANTI_BEE_DISCO.blockBuzzingSound = function()
    pcall(function()
        local beeScript = LocalPlayer.PlayerScripts:FindFirstChild("Bee", true)
        if beeScript then
            local buzzing = beeScript:FindFirstChild("Buzzing")
            if buzzing and buzzing:IsA("Sound") then
                buzzing:Stop()
                buzzing.Volume = 0
            end
        end
    end)
end

SharedState.ANTI_BEE_DISCO.Enable = function()
    local ab = SharedState.ANTI_BEE_DISCO
    if ab.running then return end
    ab.running = true
    -- Remove efeitos de iluminação
    for _, inst in ipairs(Lighting:GetDescendants()) do ab.nuke(inst) end
    table.insert(ab.connections, Lighting.DescendantAdded:Connect(function(obj)
        if not ab.running or not Config.AntiBeeDisco then return end
        ab.nuke(obj)
    end))
    -- Protege controles
    ab.protectControls()
    -- Trava FOV (COMENTADO PARA NÃO BLOQUEAR O SLIDER)
    -- ab.lockCamera()
    -- Bloqueia som do bee
    table.insert(ab.connections, RunService.Heartbeat:Connect(function()
        if not ab.running or not Config.AntiBeeDisco then return end
        ab.blockBuzzingSound()
    end))
    ShowNotification("ANTIBEE & DISCO", "Enabled")
end

SharedState.ANTI_BEE_DISCO.Disable = function()
    local ab = SharedState.ANTI_BEE_DISCO
    if not ab.running then return end
    ab.running = false
    ab.restoreControls()
    ab.restoreCamera()
    ab.disconnectAll()
    ShowNotification("ANTI-BEE & DISCO", "Disabled")
end

_G.ANTI_BEE_DISCO = SharedState.ANTI_BEE_DISCO

if Config.AntiBeeDisco then
    task.delay(1, function()
        if SharedState.ANTI_BEE_DISCO.Enable then SharedState.ANTI_BEE_DISCO.Enable() end
    end)
end
-- ============================================================-- ============================================================
-- AUTO BUY 
-- ============================================================
do -- AUTO BUY SCOPE (NOT lazy: toggleAutoBuy must exist immediately for the UI)
autoBuyActive = false
autoBuyRing = nil

function createAutoBuyRing()
    local existing = Workspace:FindFirstChild("XiAutoBuyRing")
    if existing then existing:Destroy() end
    local r = Instance.new("Part")
    r.Name = "XiAutoBuyRing"
    r.Shape = Enum.PartType.Cylinder
    r.Anchored = true
    r.CanCollide = false
    r.CanTouch = false
    r.CanQuery = false
    r.CastShadow = false
    r.Material = Enum.Material.Neon
    r.Transparency = 0.5
    r.Color = Theme.Accent
    local range = Config.AutoBuyRange or 17
    r.Size = Vector3.new(0.5, range * 2, range * 2)
    r.Parent = Workspace
    autoBuyRing = r
end

function destroyAutoBuyRing()
    if autoBuyRing then autoBuyRing:Destroy(); autoBuyRing = nil end
    local e = Workspace:FindFirstChild("XiAutoBuyRing")
    if e then e:Destroy() end
end

_abRingFrame = 0
RunService.Heartbeat:Connect(function()
    if not autoBuyActive then return end
    _abRingFrame = _abRingFrame + 1
    if _abRingFrame < 3 then return end
    _abRingFrame = 0
    local char = player.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp or not autoBuyRing then return end
    local range = Config.AutoBuyRange or 17
    autoBuyRing.Size = Vector3.new(0.5, range * 2, range * 2)
    autoBuyRing.CFrame = (hrp.CFrame * CFrame.Angles(0, 0, math.rad(90))) + Vector3.new(0, -2.5, 0)
end)

-- Explicit assignment to the OUTER `local toggleAutoBuy` declared at line ~6899.
-- Was previously `function toggleAutoBuy(on)` -- syntactic sugar that some
-- contexts can mis-resolve; this form guarantees we update the outer upvalue
-- the UI closure at line 8131 captures.
toggleAutoBuy = function(on)
    if on ~= nil then
        autoBuyActive = on
    else
        autoBuyActive = not autoBuyActive
    end
    Config.AutoBuyEnabled = autoBuyActive
    pcall(saveConfig)
    pcall(setToggle, "Auto Buy", autoBuyActive)
    if autoBuyActive then createAutoBuyRing() else destroyAutoBuyRing() end
    pcall(ShowNotification, "AUTO BUY", autoBuyActive and "ENABLED" or "DISABLED")
    if _G.AutoBuyOnToggle then
        pcall(_G.AutoBuyOnToggle, autoBuyActive)
    end
end

RARITY_WORDS = {
    common = true, uncommon = true, rare = true, epic = true,
    legendary = true, secret = true, divine = true, rainbow = true,
    cursed = true, gold = true, diamond = true,
}

function getBrainrotName(model)
    if not model then return "Brainrot", "" end
    local nameFound, genFound = "", ""
    for _, bb in ipairs(model:GetDescendants()) do
        if bb:IsA("BillboardGui") then
            for _, lbl in ipairs(bb:GetDescendants()) do
                if lbl:IsA("TextLabel") and lbl.Text and lbl.Text ~= "" then
                    local t = lbl.Text:match("^%s*(.-)%s*$")
                    local tl = t:lower()
                    if RARITY_WORDS[tl] then continue end
                    if t:match("^%$[%d%.]+[KkMmBb]?/s$") then
                        if genFound == "" then genFound = t end
                        continue
                    end
                    if t:match("^%$[%d%.]+[KkMmBb]?$") then continue end
                    if t:match("^[%d%.]+[KkMmBb]?$") then continue end
                    if nameFound == "" and #t > 1 then nameFound = t end
                end
            end
        end
    end
    if nameFound == "" then
        pcall(function()
            local info = AnimalsData[model.Name]
            if info and info.DisplayName then
                nameFound = info.DisplayName
                local gv = AnimalsShared:GetGeneration(model.Name, nil, nil, nil)
                local gt = "$" .. NumberUtils:ToString(gv) .. "/s"
                genFound = gt
            end
        end)
    end
    if nameFound == "" then nameFound = model.Name ~= "" and model.Name or "Brainrot" end
    return nameFound, genFound
end

function scanConveyor()
    local results = {}
    for _, obj in ipairs(Workspace:GetDescendants()) do
        if not (obj:IsA("ProximityPrompt") and obj.Enabled) then continue end
        local txt = obj.ActionText or ""
        if not (txt == "Purchase" or txt:lower():find("purchase") or txt:lower():find("comprar")) then continue end
        local part = obj.Parent
        if not part then continue end
        local realPart = (part:IsA("Attachment") and part.Parent) or part
        if not (realPart and realPart:IsA("BasePart")) then continue end
        local model, cur = nil, realPart
        for _ = 1, 8 do
            if cur and cur:IsA("Model") then model = cur; break end
            cur = cur and cur.Parent
        end
        local name, gen = getBrainrotName(model)
        table.insert(results, {
            name = name,
            gen = gen,
            prompt = obj,
            part = realPart,
            model = model,
            source = "ESTEIRA",
            uid = "esteira_" .. tostring(obj),
        })
    end
    return results
end

SharedState.ConveyorAnimals = {}
function refreshConveyor()
    local ok, found = pcall(scanConveyor)
    if ok and found then
        SharedState.ConveyorAnimals = found
    end
end
refreshConveyor()
_G.refreshConveyor = refreshConveyor

purchaseRemote = nil
function resolvePurchaseRemote()
    if purchaseRemote and purchaseRemote.Parent then return purchaseRemote end
    pcall(function()
        local paths = {"RF/ShopService/BuyAnimal", "RF/AnimalShop/Purchase", "RE/Shop/Buy", "RF/Shop/Buy"}
        for _, p in ipairs(paths) do
            local ok2, r = pcall(function() return Decrypted[p] end)
            if ok2 and r and r.Parent then
                purchaseRemote = r
                return
            end
        end
    end)
    return purchaseRemote
end

function firePurchaseNatural(prompt)
    if not prompt or not prompt.Parent or not prompt.Enabled then return end
    pcall(function()
        if fireproximityprompt then fireproximityprompt(prompt) end
    end)
    task.spawn(function()
        local remote = resolvePurchaseRemote()
        if remote then
            pcall(function()
                if remote:IsA("RemoteFunction") then
                    remote:InvokeServer(prompt.Parent)
                elseif remote:IsA("RemoteEvent") then
                    remote:FireServer(prompt.Parent)
                end
            end)
        end
    end)
end

carpetLockConn = nil
function startCarpetLock()
    if carpetLockConn then carpetLockConn:Disconnect(); carpetLockConn = nil end
    local function ensureCarpet()
        pcall(function()
            local char = LocalPlayer.Character
            local hum = char and char:FindFirstChildOfClass("Humanoid")
            if not hum then return end
            local toolName = (Config.TpSettings and Config.TpSettings.Tool) or "Flying Carpet"
            if not char:FindFirstChild(toolName) then
                local tool = LocalPlayer.Backpack:FindFirstChild(toolName)
                if tool then hum:EquipTool(tool) end
            end
        end)
    end
    task.spawn(function()
        for _ = 1, 15 do
            if not autoBuyActive then break end
            ensureCarpet()
            task.wait(0.3)
            local char = LocalPlayer.Character
            local toolName = (Config.TpSettings and Config.TpSettings.Tool) or "Flying Carpet"
            if char and char:FindFirstChild(toolName) then break end
        end
    end)
    carpetLockConn = RunService.Heartbeat:Connect(function()
        if not autoBuyActive then return end
        ensureCarpet()
    end)
end

function stopCarpetLock()
    if carpetLockConn then carpetLockConn:Disconnect(); carpetLockConn = nil end
end

HOVER_HEIGHT = 5
BUY_INTERVAL = 0.08
DETECT_RADIUS = 17
lockedTarget = nil
lockedPart = nil
lockedModel = nil

function partAlive()
    return lockedPart and lockedPart.Parent and lockedModel and lockedModel.Parent
end

function promptAlive()
    return lockedTarget and lockedTarget.prompt and lockedTarget.prompt.Parent and lockedTarget.prompt.Enabled
end

bodyPos = nil
function ensureBodyPos(hrp)
    if bodyPos and bodyPos.Parent == hrp then
        local speed = math.clamp(Config.AutoGrabSpeed or 17, 5, 100)
        bodyPos.P = speed * 1000
        return bodyPos
    end
    if bodyPos then bodyPos:Destroy() end
    local speed = math.clamp(Config.AutoGrabSpeed or 17, 5, 100)
    local bp = Instance.new("BodyPosition", hrp)
    bp.MaxForce = Vector3.new(math.huge, math.huge, math.huge)
    bp.P = speed * 1000
    bp.D = 1000
    bp.Position = hrp.Position
    bodyPos = bp
    return bp
end

function destroyBodyPos()
    if bodyPos then bodyPos:Destroy(); bodyPos = nil end
end

RunService.Heartbeat:Connect(function()
    if not autoBuyActive or not partAlive() then
        destroyBodyPos()
        return
    end
    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then destroyBodyPos(); return end
    local above = lockedPart.Position + Vector3.new(0, HOVER_HEIGHT, 0)
    local bp = ensureBodyPos(hrp)
    bp.Position = above
end)

task.spawn(function()
    while true do
        task.wait(BUY_INTERVAL)
        if not autoBuyActive then continue end
        if not partAlive() then continue end
        if promptAlive() then
            firePurchaseNatural(lockedTarget.prompt)
        end
    end
end)

task.spawn(function()
    while true do
        task.wait(0.25)
        if not autoBuyActive then
            lockedTarget = nil
            lockedPart = nil
            lockedModel = nil
            stopCarpetLock()
            destroyBodyPos()
            continue
        end
        if lockedPart or lockedModel then
            if not partAlive() then
                ShowNotification("AUTO BUY", "Reached base, scanning...")
                pcall(refreshConveyor)
                lockedTarget = nil
                lockedPart = nil
                lockedModel = nil
            end
            continue
        end
        pcall(refreshConveyor)
        local char = LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then continue end
        local radius = Config.AutoBuyRange or DETECT_RADIUS
        local best, bestDist = nil, math.huge
        for _, entry in ipairs(SharedState.ConveyorAnimals) do
            if entry.prompt and entry.prompt.Parent and entry.prompt.Enabled and entry.part and entry.part.Parent then
                local d = (hrp.Position - entry.part.Position).Magnitude
                if d <= radius and d < bestDist then
                    bestDist = d
                    best = entry
                end
            end
        end
        if best then
            lockedTarget = best
            lockedPart = best.part
            lockedModel = best.model or best.part.Parent
            ShowNotification("AUTO BUY", "Locked: " .. best.name)
            startCarpetLock()
        end
    end
end)

_G.AutoBuyOnToggle = function(active)
    if active then
        if _G.refreshConveyor then pcall(_G.refreshConveyor) end
        startCarpetLock()
    else
        stopCarpetLock()
        destroyBodyPos()
    end
end
end -- END AUTO BUY SCOPE

-- ============================================================
-- ============================================================

function hasExclamation(target)
    for _, d in ipairs(target:GetDescendants()) do
        if d:IsA("BillboardGui") then
            -- check EVERY label: the "!" isn't always in the first TextLabel
            for _, l in ipairs(d:GetDescendants()) do
                if (l:IsA("TextLabel") or l:IsA("TextBox")) and tostring(l.Text):find("!") then return true end
            end
        end
    end
    return false
end

do -- AUTO DESTROY TURRETS (do-block: locals stay off the main chunk's register cap)
    local sentryCache, lastSentryScan = {}, 0
    local swinging = false
    local function findBat(char)
        local function scan(cont)
            if not cont then return nil end
            for _, t in ipairs(cont:GetChildren()) do
                if t:IsA("Tool") and t.Name:lower():find("bat", 1, true) then return t end
            end
        end
        return scan(char) or scan(player:FindFirstChild("Backpack"))
    end
    task.spawn(function()
        while true do task.wait(0.1)
            -- Paused while stealing/carrying or mid-TP (equipping the bat there
            -- would drop the brainrot / break the flight) -- otherwise always on
            if Config.AutoDestroyTurrets and LocalPlayer:GetAttribute("Stealing") ~= true
                and not _G._isTpMoving then
                local char = LocalPlayer.Character
                local hrp = char and char:FindFirstChild("HumanoidRootPart")
                local hum = char and char:FindFirstChildOfClass("Humanoid")
                if hrp and hum and hum.Health > 0 then
                    -- sentry list refreshed 1x/s (full-workspace GetDescendants at
                    -- 10x/s was an FPS sink); distance check stays at 10x/s
                    local now = os.clock()
                    if now - lastSentryScan > 1 then
                        lastSentryScan = now
                        sentryCache = {}
                        for _, inst in ipairs(Workspace:GetDescendants()) do
                            if inst.Name:match("^Sentry_") then sentryCache[#sentryCache + 1] = inst end
                        end
                    end
                    -- INFINITE RANGE: every sentry on the map is a target, aggro'd
                    -- ("!") or not -- alerted ones first, then the nearest one
                    local closest, shortestDist = nil, math.huge
                    local closestCalm, shortestCalm = nil, math.huge
                    for _, inst in ipairs(sentryCache) do
                        if inst.Parent then
                            local root = inst:IsA("BasePart") and inst or inst:FindFirstChildWhichIsA("BasePart", true)
                            if root then
                                local dist = (hrp.Position - root.Position).Magnitude
                                if hasExclamation(inst) then
                                    if dist < shortestDist then shortestDist = dist; closest = inst end
                                else
                                    if dist < shortestCalm then shortestCalm = dist; closestCalm = inst end
                                end
                            end
                        end
                    end
                    if not closest then closest = closestCalm end
                    if closest and not swinging then
                        swinging = true
                        task.spawn(function()
                            pcall(function()
                                -- make the turret visible + pull it in front of the player
                                for _, d in ipairs(closest:GetDescendants()) do
                                    if d:IsA("BasePart") then d.Transparency = 0.5; d.CanCollide = false end
                                end
                                local targetCF = CFrame.new(hrp.Position + hrp.CFrame.LookVector * 4, hrp.Position)
                                if closest:IsA("Model") then closest:PivotTo(targetCF)
                                elseif closest:IsA("BasePart") then closest.CFrame = targetCF end
                                local bat = findBat(char)
                                if bat then
                                    if bat.Parent ~= char then
                                        -- RE-EQUIP LOOP: invisible steal / TP tool
                                        -- managers can unequip us -- keep retrying
                                        -- for up to 1s instead of a single attempt
                                        local t0 = os.clock()
                                        while bat.Parent ~= char and os.clock() - t0 < 1 do
                                            pcall(function() hum:EquipTool(bat) end)
                                            task.wait(0.08)
                                        end
                                    end
                                    -- several swings: a single one often misses while
                                    -- the repositioned turret settles
                                    for _ = 1, 4 do
                                        if not (closest and closest.Parent) then break end
                                        if bat.Parent ~= char then pcall(function() hum:EquipTool(bat) end) end
                                        pcall(function() bat:Activate() end)
                                        task.wait(0.12)
                                    end
                                end
                            end)
                            swinging = false
                        end)
                    end
                end
            end
        end
    end)
end
-- REMOTE SELL (removed from UI)
pcall(function()
    local old = playerGui:FindFirstChild("Mynxx_RemoteSell")
    if old then old:Destroy() end
end)
_G.toggleRemoteSell = function() end

-- ============================================================
-- GUI HELPERS
-- ============================================================
main,mainBody,tabBar,sidebarNav,sidebarProfile,bottomBar,fpsText = nil,nil,nil,nil,nil,nil,nil
panels,panelSetters,tabButtons={},{},{}
actionSettingsPanel,actionSettingsBody,tpSpeedSettingsPanel,tpSpeedSettingsBody = nil,nil,nil,nil
BoundToggles={}

stealProgressBarGui = nil

-- Pre-declare helper functions made global to free local registers in outer scope

_G.ShowStealProgressBar = function(targetName, duration)
    local ExploitGui = (gethui and gethui()) or game:GetService("CoreGui")
    
    if stealProgressBarGui then
        pcall(function() stealProgressBarGui:Destroy() end)
    end
    
    local sg = Instance.new("ScreenGui")
    sg.Name = "Mynxx_StealProgressBar"
    sg.ResetOnSpawn = false
    sg.Parent = ExploitGui
    stealProgressBarGui = sg
    
    local barWidth = 270
    local barHeight = 50
    
    local container = Instance.new("Frame")
    container.Size = UDim2.fromOffset(barWidth, barHeight)
    container.Position = UDim2.new(0.5, -barWidth/2, 1, -190)
    container.BackgroundColor3 = Theme.Background
    container.BackgroundTransparency = UITransparency.Progress
    container.BorderSizePixel = 0
    container.Parent = registerScreenGui(sg)
    corner(container, 12)
    addOutline(container)
    
    local title = Instance.new("TextLabel")
    title.Size = UDim2.new(0.6, -14, 0, 16)
    title.Position = UDim2.new(0, 14, 0, 8)
    title.BackgroundTransparency = 1
    title.Text = "⚡ STEALING: " .. (targetName or "Brainrot"):upper()
    title.TextColor3 = Theme.Text
    title.Font = Enum.Font.GothamBold
    title.TextSize = 9.5
    title.TextXAlignment = Enum.TextXAlignment.Left
    title.Parent = container
    
    local pctLabel = Instance.new("TextLabel")
    pctLabel.Size = UDim2.new(0.4, -14, 0, 16)
    pctLabel.Position = UDim2.new(0.6, 0, 0, 8)
    pctLabel.BackgroundTransparency = 1
    pctLabel.Text = "0%"
    pctLabel.TextColor3 = Theme.Text
    pctLabel.Font = Enum.Font.GothamBold
    pctLabel.TextSize = 10
    pctLabel.TextXAlignment = Enum.TextXAlignment.Right
    pctLabel.Parent = container
    
    local track = Instance.new("Frame")
    track.Size = UDim2.new(1, -28, 0, 6)
    track.Position = UDim2.new(0, 14, 0, 30)
    track.BackgroundColor3 = Theme.SliderBg
    track.BorderSizePixel = 0
    track.Parent = container
    corner(track, 3)
    stroke(track, Theme.Stroke, 1, 0.1)
    
    local fill = Instance.new("Frame")
    fill.Size = UDim2.new(0, 0, 1, 0)
    fill.BackgroundColor3 = Color3.fromRGB(255, 255, 255)
    fill.BorderSizePixel = 0
    fill.Parent = track
    corner(fill, 3)
    
    local fillGrad = Instance.new("UIGradient", fill)
    fillGrad.Color = ColorSequence.new({
        ColorSequenceKeypoint.new(0, Color3.fromRGB(255, 255, 255)),
        ColorSequenceKeypoint.new(1, Color3.fromRGB(200, 200, 208)),
    })
    
    local safeDuration = tonumber(duration) or 5
    if safeDuration <= 0.01 then safeDuration = 5 end
    local startTick = tick()
    
    task.spawn(function()
        while sg and sg.Parent do
            local elapsed = tick() - startTick
            local t = math.clamp(elapsed / safeDuration, 0, 1)
            local ratio = 1 - (1 - t)^3
            
            pcall(function()
                pctLabel.Text = math.floor(ratio * 100) .. "%"
                fill.Size = UDim2.new(ratio, 0, 1, 0)
            end)
            
            if t >= 1 then break end
            task.wait()
        end
    end)
end

_G.HideStealProgressBar = function()
    local targetGui = stealProgressBarGui
    if targetGui then
        stealProgressBarGui = nil
        pcall(function()
            local container = targetGui:FindFirstChildWhichIsA("Frame")
            if container then
                tw(container, {BackgroundTransparency = 1}, 0.15)
                for _, child in ipairs(container:GetDescendants()) do
                    if child:IsA("TextLabel") then
                        tw(child, {TextTransparency = 1}, 0.15)
                    elseif child:IsA("Frame") then
                        tw(child, {BackgroundTransparency = 1}, 0.15)
                    elseif child:IsA("UIStroke") then
                        tw(child, {Transparency = 1}, 0.15)
                    end
                end
                task.wait(0.16)
            end
            targetGui:Destroy()
        end)
    end
end

-- Assign helper implementations to pre-declared outer local variables
corner = function(o,r) local c=Instance.new("UICorner"); c.CornerRadius=UDim.new(0,r); c.Parent=o; return c end
stroke = function(o,col,th,tr) local s=Instance.new("UIStroke"); s.Color=col or Theme.Stroke; s.Thickness=th or 1; s.Transparency=tr or 0; s.ApplyStrokeMode=Enum.ApplyStrokeMode.Border; s.Parent=o; return s end
tw = function(o,p,t) TweenService:Create(o,TweenInfo.new(t or 0.14,Enum.EasingStyle.Quint,Enum.EasingDirection.Out),p):Play() end
addOutline = function(f) local o=Instance.new("UIStroke"); o.Color=Theme.Stroke; o.Thickness=1; o.Transparency=UITransparency.Outline; o.ApplyStrokeMode=Enum.ApplyStrokeMode.Border; o.Parent=f; return o end
function clearBody(body) for _,c in ipairs(body:GetChildren()) do if not c:IsA("UIListLayout") and not c:IsA("UIPadding") then c:Destroy() end end end

function openAnim(f) if not f then return end; local us=f:FindFirstChild("MynxxScale") or Instance.new("UIScale"); us.Name="MynxxScale"; us.Parent=f
    local tgt=f.Position; f.Visible=true; us.Scale=0.92; f.Position=UDim2.new(tgt.X.Scale,tgt.X.Offset,tgt.Y.Scale,tgt.Y.Offset+18); tw(us,{Scale=1},0.20); tw(f,{Position=tgt},0.20) end
function closeAnim(f) if not f then return end; f.Visible = false end

makeDraggable = function(frame,handle,saveName) local dragging,dragStart,startPos=false,nil,nil
    handle.InputBegan:Connect(function(i) if UI.Locked then return end; if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then dragging=true; dragStart=i.Position; startPos=frame.Position end end)
    UIS.InputEnded:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then if dragging and saveName then rememberPosition(saveName,frame) end; dragging=false end end)
    UIS.InputChanged:Connect(function(i) if dragging and not UI.Locked and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then
        local d=i.Position-dragStart
        local scale = getGlobalScale()
        frame.Position=UDim2.new(startPos.X.Scale,startPos.X.Offset+(d.X/scale),startPos.Y.Scale,startPos.Y.Offset+(d.Y/scale))
    end end)
end

makeResizable = function(frame, minSize, panelName)
    local h = Instance.new("TextButton")
    h.Name = "ResizeGrip"
    h.Size = UDim2.new(0, 16, 0, 16)
    h.Position = UDim2.new(1, -16, 1, -16)
    h.BackgroundTransparency = 1
    h.Text = "◢"
    h.TextColor3 = Theme.Dim or Color3.new(1, 1, 1)
    h.TextSize = 12
    h.ZIndex = 100
    h.Visible = not UI.Locked -- grip only shows while the UI is unlocked
    h.Parent = frame
    _G.__MynxxGrips = _G.__MynxxGrips or {}
    table.insert(_G.__MynxxGrips, h)
    local dragging, dragStart, startSize = false, nil, nil
    h.InputBegan:Connect(function(i)
        if UI.Locked then return end
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
            dragging = true; dragStart = i.Position; startSize = frame.AbsoluteSize
        end
    end)
    UIS.InputEnded:Connect(function(i)
        if i.UserInputType == Enum.UserInputType.MouseButton1 or i.UserInputType == Enum.UserInputType.Touch then
            if dragging then
                dragging = false
                if panelName then
                    if not Config.sizes then Config.sizes = {} end
                    Config.sizes[panelName] = {x = frame.Size.X.Offset, y = frame.Size.Y.Offset}
                    saveConfig()
                end
            end
        end
    end)
    UIS.InputChanged:Connect(function(i)
        if dragging and not UI.Locked and (i.UserInputType == Enum.UserInputType.MouseMovement or i.UserInputType == Enum.UserInputType.Touch) then
            local d = i.Position - dragStart
            local scale = getGlobalScale()
            local nx = math.max(minSize.X.Offset, startSize.X + (d.X / scale))
            local ny = math.max(minSize.Y.Offset, startSize.Y + (d.Y / scale))
            frame.Size = UDim2.new(0, nx, 0, ny)
        end
    end)
end

function makeHeader(f,t,isMain) local h=Instance.new("Frame"); h.Size=UDim2.new(1,0,0,42); h.BackgroundTransparency=1; h.Parent=f
    local parts={}; for s in string.gmatch(t,"([^\n]+)") do table.insert(parts,s) end
    if isMain then local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-50,0,24); l.Position=UDim2.new(0,13,0,8); l.BackgroundTransparency=1; l.Text=parts[1] or "Sxe Remasterd"; l.TextColor3=Theme.Text; l.Font=UIFont.Title; l.TextSize=16; l.TextXAlignment=Enum.TextXAlignment.Left; l.Parent=h
    elseif parts[2] == "Invisible Steal" then
        local brand=Instance.new("TextLabel"); brand.Size=UDim2.new(1,-24,0,14); brand.Position=UDim2.new(0,12,0,5); brand.BackgroundTransparency=1; brand.Text=parts[1] or "Sxe Remasterd"; brand.TextColor3=Theme.Text; brand.Font=Enum.Font.GothamBlack; brand.TextSize=11; brand.TextXAlignment=Enum.TextXAlignment.Center; brand.Parent=h
        local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-24,0,16); l.Position=UDim2.new(0,12,0,20); l.BackgroundTransparency=1; l.Text=parts[2]; l.TextColor3=Theme.Text; l.Font=Enum.Font.GothamBold; l.TextSize=12; l.TextXAlignment=Enum.TextXAlignment.Center; l.Parent=h
    else
        local brand=Instance.new("TextLabel"); brand.Size=UDim2.new(1,-58,0,12); brand.Position=UDim2.new(0,12,0,5); brand.BackgroundTransparency=1; brand.Text=parts[1] or "Sxe Remasterd"; brand.TextColor3=Theme.Dim; brand.Font=UIFont.Brand; brand.TextSize=9; brand.TextXAlignment=Enum.TextXAlignment.Center; brand.Parent=h
        local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-58,0,16); l.Position=UDim2.new(0,12,0,18); l.BackgroundTransparency=1; l.Text=parts[2] or ""; l.TextColor3=Theme.Text; l.Font=UIFont.Title; l.TextSize=12; l.TextXAlignment=Enum.TextXAlignment.Center; l.Parent=h
    end
    local d=Instance.new("Frame"); d.Size=UDim2.new(1,-24,0,1); d.Position=UDim2.new(0,12,0,40); d.BackgroundColor3=Theme.Stroke; d.BackgroundTransparency=0.55; d.BorderSizePixel=0; d.Parent=f
    makeDraggable(f,h,t); return h end

function makeMainPanel(t,size,pos)
    local f=Instance.new("Frame"); f.Size=size; f.Position=pos; f.BackgroundColor3=Theme.MainBackground; f.BackgroundTransparency=UITransparency.MainPanel; f.BorderSizePixel=0; f.ClipsDescendants=true; f.Parent=gui; corner(f,14); addOutline(f)
    -- TOP HEADER (drag area + title + profile)
    local dragHandle=Instance.new("Frame"); dragHandle.Name="DragHandle"; dragHandle.Size=UDim2.new(1,0,0,44); dragHandle.BackgroundColor3=Theme.SidebarBg or Color3.fromRGB(0,0,0); dragHandle.BackgroundTransparency=UITransparency.Sidebar; dragHandle.BorderSizePixel=0; dragHandle.Parent=f; corner(dragHandle,14)
    -- cover the bottom rounded corners of header so it looks flat on bottom
    local hdrFill=Instance.new("Frame"); hdrFill.Size=UDim2.new(1,0,0,14); hdrFill.Position=UDim2.new(0,0,1,-14); hdrFill.BackgroundColor3=dragHandle.BackgroundColor3; hdrFill.BackgroundTransparency=dragHandle.BackgroundTransparency; hdrFill.BorderSizePixel=0; hdrFill.Parent=dragHandle
    local titleParts={}; for s in string.gmatch(t or "Sxe Remasterd","([^\n]+)") do table.insert(titleParts,s) end
    local titleLbl=Instance.new("TextLabel"); titleLbl.Name="SidebarTitle"; titleLbl.Size=UDim2.new(1,-110,1,0); titleLbl.Position=UDim2.new(0,16,0,0); titleLbl.BackgroundTransparency=1; titleLbl.Text=titleParts[1] or "Sxe Remasterd"; titleLbl.TextColor3=Theme.Text; titleLbl.Font=Enum.Font.GothamBlack; titleLbl.TextSize=13; titleLbl.TextXAlignment=Enum.TextXAlignment.Left; titleLbl.TextYAlignment=Enum.TextYAlignment.Center; titleLbl.Parent=dragHandle
    -- Profile (avatar + name) compacted on the right of header
    -- HORIZONTAL TAB BAR below header
    local tabBarFrame=Instance.new("Frame"); tabBarFrame.Name="TopTabBar"; tabBarFrame.Size=UDim2.new(1,0,0,34); tabBarFrame.Position=UDim2.new(0,0,0,44); tabBarFrame.BackgroundColor3=Theme.SidebarBg or Color3.fromRGB(0,0,0); tabBarFrame.BackgroundTransparency=(UITransparency.Sidebar or 0)+0.1; tabBarFrame.BorderSizePixel=0; tabBarFrame.ClipsDescendants=true; tabBarFrame.Parent=f
    local tabBarScroll=Instance.new("ScrollingFrame"); tabBarScroll.Size=UDim2.new(1,-8,1,0); tabBarScroll.Position=UDim2.new(0,4,0,0); tabBarScroll.BackgroundTransparency=1; tabBarScroll.BorderSizePixel=0; tabBarScroll.ScrollBarThickness=0; tabBarScroll.CanvasSize=UDim2.new(0,0,0,0); tabBarScroll.ScrollingDirection=Enum.ScrollingDirection.X; tabBarScroll.Parent=tabBarFrame
    local tabLay=Instance.new("UIListLayout"); tabLay.FillDirection=Enum.FillDirection.Horizontal; tabLay.Padding=UDim.new(0,3); tabLay.SortOrder=Enum.SortOrder.LayoutOrder; tabLay.VerticalAlignment=Enum.VerticalAlignment.Center; tabLay.Parent=tabBarScroll
    tabLay:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function() tabBarScroll.CanvasSize=UDim2.new(0,tabLay.AbsoluteContentSize.X+8,0,0) end)
    sidebarNav=tabBarScroll
    -- Thin separator between tab bar and content
    local sep=Instance.new("Frame"); sep.Size=UDim2.new(1,0,0,1); sep.Position=UDim2.new(0,0,0,78); sep.BackgroundColor3=Theme.Stroke; sep.BackgroundTransparency=0.45; sep.BorderSizePixel=0; sep.Parent=f
    -- CONTENT AREA (full width, below tab bar)
    local contentBg=Instance.new("Frame"); contentBg.Name="ContentBg"; contentBg.Size=UDim2.new(1,-10,1,-88); contentBg.Position=UDim2.new(0,5,0,80); contentBg.BackgroundColor3=Theme.ContentBg or Theme.MainBackground; contentBg.BackgroundTransparency=UITransparency.Content; contentBg.BorderSizePixel=0; contentBg.Parent=f; corner(contentBg,12)
    local content=Instance.new("Frame"); content.Name="Content"; content.Size=UDim2.new(1,0,1,0); content.BackgroundTransparency=1; content.Parent=contentBg
    local body=Instance.new("ScrollingFrame"); body.Size=UDim2.new(1,-20,1,-16); body.Position=UDim2.new(0,10,0,8); body.BackgroundTransparency=1; body.BorderSizePixel=0; body.ScrollBarThickness=2; body.ScrollBarImageColor3=Theme.Stroke; body.ScrollBarImageTransparency=0.35; body.CanvasSize=UDim2.new(0,0,0,0); body.Active=true; body.Parent=content
    local pad=Instance.new("UIPadding"); pad.PaddingTop=UDim.new(0,6); pad.PaddingBottom=UDim.new(0,10); pad.PaddingLeft=UDim.new(0,2); pad.PaddingRight=UDim.new(0,2); pad.Parent=body
    local lay=Instance.new("UIListLayout"); lay.Padding=UDim.new(0,8); lay.Parent=body
    lay:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function() body.CanvasSize=UDim2.new(0,0,0,lay.AbsoluteContentSize.Y+16) end)
    tabBar=sidebarNav
    if Config.sizes and Config.sizes[t] then f.Size=UDim2.new(0,Config.sizes[t].x,0,Config.sizes[t].y) end
    makeDraggable(f,dragHandle,t); makeResizable(f,UDim2.new(0,340,0,300),t); return f,body
end

function makeQuickPanel(t,size,pos) local f=Instance.new("Frame"); f.Size=size; f.Position=pos; f.BackgroundColor3=Theme.Background; f.BackgroundTransparency=UITransparency.Panel; f.BorderSizePixel=0; f.ClipsDescendants=true; f.Parent=gui; corner(f,12); addOutline(f); makeHeader(f,t,false)
    local body=Instance.new("ScrollingFrame"); body.Size=UDim2.new(1,-12,1,-50); body.Position=UDim2.new(0,6,0,46); body.BackgroundTransparency=1; body.BorderSizePixel=0; body.ScrollBarThickness=2; body.ScrollBarImageColor3=Theme.Stroke; body.ScrollBarImageTransparency=0.4; body.CanvasSize=UDim2.new(0,0,0,0); body.Active=true; body.Parent=f
    local lay=Instance.new("UIListLayout"); lay.Padding=UDim.new(0,6); lay.Parent=body
    lay:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function() body.CanvasSize=UDim2.new(0,0,0,lay.AbsoluteContentSize.Y+10) end); 
    if Config.sizes and Config.sizes[t] then f.Size = UDim2.new(0, Config.sizes[t].x, 0, Config.sizes[t].y) end
    if not string.find(t, "Admin Command Panel") then makeResizable(f, UDim2.new(0, 150, 0, 150), t) end; return f,body end

function makeSyncStateRow(parent,text,toggleName,callback)
    regToggle(toggleName,getToggle(toggleName))
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,31); row.BackgroundTransparency=1; row.Parent=parent
    local label=Instance.new("TextLabel"); label.Size=UDim2.new(1,-62,1,0); label.Position=UDim2.new(0,4,0,0); label.BackgroundTransparency=1; label.Text=text; label.TextColor3=Theme.Text; label.Font=UIFont.Label; label.TextSize=11; label.TextXAlignment=Enum.TextXAlignment.Left; label.TextTruncate=Enum.TextTruncate.AtEnd; label.Parent=row
    local btn=Instance.new("TextButton"); btn.Name="SyncStateBtn"; btn.Size=UDim2.new(0,52,0,24); btn.Position=UDim2.new(1,-56,0.5,-12); btn.Font=UIFont.Button; btn.TextSize=11; btn.AutoButtonColor=false; btn.Parent=row; corner(btn,6)
    local function refresh(val)
        btn.BackgroundColor3=val and (Theme.ToggleOn or Color3.fromRGB(245,245,247)) or Theme.ToggleOff
        btn.BackgroundTransparency=UITransparency.Toggle
        btn.Text=val and "ON" or "OFF"
        btn.TextColor3=Color3.new(1,1,1)
    end
    refresh(getToggle(toggleName)); onToggleChanged(toggleName,function(val) refresh(val) end)
    btn.MouseButton1Click:Connect(function() local nv=not getToggle(toggleName); setToggle(toggleName,nv); if callback then callback(nv) end end)
    return function(ns,fire) if typeof(ns)=="boolean" then setToggle(toggleName,ns); if fire~=false and callback then callback(ns) end end end, label
end

function makeSyncMainToggle(parent,text,toggleName,callback)
    regToggle(toggleName,getToggle(toggleName))
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,32); row.BackgroundTransparency=1; row.Parent=parent
    local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-34,1,0); l.Position=UDim2.new(0,2,0,0); l.BackgroundTransparency=1; l.Text=text; l.TextColor3=Theme.Text; l.Font=UIFont.Label; l.TextSize=12; l.TextXAlignment=Enum.TextXAlignment.Left; l.TextTruncate=Enum.TextTruncate.AtEnd; l.Parent=row
    -- Nexus checkbox style v2: small, sharp, outline when off / flat blue when on
    local toggle=Instance.new("TextButton"); toggle.Size=UDim2.new(0,15,0,15); toggle.Position=UDim2.new(1,-21,0.5,-8); toggle.Text=""; toggle.AutoButtonColor=false; toggle.Parent=row; corner(toggle,3)
    stroke(toggle,Theme.Stroke,1,0)
    local tStroke=toggle:FindFirstChildOfClass("UIStroke")
    local check=Instance.new("TextLabel"); check.Size=UDim2.new(1,0,1,0); check.Position=UDim2.new(0,0,0,-1); check.BackgroundTransparency=1; check.Text="✓"; check.TextColor3=Theme.ToggleKnobOn or Color3.new(1,1,1); check.Font=Enum.Font.GothamBold; check.TextSize=10; check.Parent=toggle
    local hit=Instance.new("TextButton"); hit.Size=UDim2.new(0,30,0,30); hit.Position=UDim2.new(1,-28,0.5,-15); hit.BackgroundTransparency=1; hit.Text=""; hit.Parent=row
    local function refresh(val)
        tw(toggle,{BackgroundColor3=val and (Theme.ToggleOn or Theme.Accent) or Theme.ToggleOff, BackgroundTransparency=val and 0 or 0.55},0.1)
        check.Visible=val and true or false
        if tStroke then tStroke.Color=val and (Theme.ToggleOn or Theme.Accent) or Theme.Stroke end
    end
    refresh(getToggle(toggleName))
    toggle.BackgroundColor3=getToggle(toggleName) and (Theme.ToggleOn or Theme.Accent) or Theme.ToggleOff
    toggle.BackgroundTransparency=getToggle(toggleName) and 0 or 0.55
    onToggleChanged(toggleName,function(val) refresh(val) end)
    local function clickFlip() local nv=not getToggle(toggleName); setToggle(toggleName,nv); if callback then callback(nv) end end
    toggle.MouseButton1Click:Connect(clickFlip)
    hit.MouseButton1Click:Connect(clickFlip)
    BoundToggles[text]=function(ns,fire) if typeof(ns)=="boolean" then setToggle(toggleName,ns); if fire~=false and callback then callback(ns) end end end
    return BoundToggles[text]
end

function makeQuickButton(parent,text,callback,bg) local b=Instance.new("TextButton"); b.Size=UDim2.new(1,-4,0,36); b.BackgroundColor3=bg or Theme.SoftButton; b.BackgroundTransparency=UITransparency.Button; b.Text=text; b.TextColor3=Theme.Text; b.Font=UIFont.Button; b.TextSize=13; b.AutoButtonColor=false; b.Parent=parent; corner(b,6)
    b.MouseEnter:Connect(function() tw(b,{BackgroundColor3=bg or Theme.SoftButtonHover},0.12) end); b.MouseLeave:Connect(function() tw(b,{BackgroundColor3=bg or Theme.SoftButton},0.12) end)
    b.MouseButton1Click:Connect(function() if callback then callback() end end); return b end

function makeQuickSlider(parent,text,min,max,default,callback,suffix) local holder=Instance.new("Frame"); holder.Size=UDim2.new(1,-4,0,54); holder.BackgroundTransparency=1; holder.Parent=parent
    local label=Instance.new("TextLabel"); label.Size=UDim2.new(1,0,0,18); label.Position=UDim2.new(0,2,0,0); label.BackgroundTransparency=1; label.Text=text..": "..tostring(default)..(suffix or ""); label.TextColor3=Theme.Text; label.Font=UIFont.Body; label.TextSize=12; label.TextXAlignment=Enum.TextXAlignment.Left; label.Parent=holder
    local bar=Instance.new("Frame"); bar.Size=UDim2.new(1,-8,0,4); bar.Position=UDim2.new(0,2,0,32); bar.BackgroundColor3=Theme.SliderBg or Color3.fromRGB(46,52,63); bar.BackgroundTransparency=0.25; bar.BorderSizePixel=0; bar.Parent=holder; corner(bar,6)
    local fill=Instance.new("Frame"); fill.Size=UDim2.new(math.clamp((default-min)/(max-min),0,1),0,1,0); fill.BackgroundColor3=Theme.SliderFill or Theme.Accent or Color3.fromRGB(74,140,255); fill.BackgroundTransparency=0; fill.BorderSizePixel=0; fill.Parent=bar; corner(fill,6)
    local knob=Instance.new("Frame"); knob.Size=UDim2.new(0,12,0,12); knob.AnchorPoint=Vector2.new(0.5,0.5); knob.Position=UDim2.new(math.clamp((default-min)/(max-min),0,1),0,0.5,0); knob.Name="WhiteSliderKnob"; knob.BackgroundColor3=Color3.fromRGB(255,255,255); knob.BorderSizePixel=0; knob.ZIndex=2; knob.Parent=bar; corner(knob,20)
    local dragging=false
    local function update(x) local rel=math.clamp((x-bar.AbsolutePosition.X)/bar.AbsoluteSize.X,0,1); local v=math.floor((min+(max-min)*rel)*10+0.5)/10; fill.Size=UDim2.new(rel,0,1,0); knob.Position=UDim2.new(rel,0,0.5,0); label.Text=text..": "..tostring(v)..(suffix or ""); if callback then callback(v) end end
    bar.InputBegan:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then dragging=true; update(i.Position.X) end end)
    UIS.InputEnded:Connect(function(i) if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then dragging=false end end)
    UIS.InputChanged:Connect(function(i) if dragging and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then update(i.Position.X) end end)
    local function setVal(v, silent)
        v = math.clamp(v, min, max)
        local rel = (v - min) / (max - min)
        local displayVal = math.floor(v * 10 + 0.5) / 10
        fill.Size = UDim2.new(rel, 0, 1, 0)
        knob.Position = UDim2.new(rel, 0, 0.5, 0)
        label.Text = text..": "..tostring(displayVal)..(suffix or "")
        if callback and not silent then callback(displayVal) end
    end
    return {Set = setVal}
end

function makeMainSliderWithInput(parent,text,min,max,default,callback,suffix)
    local holder=Instance.new("Frame")
    holder.Size=UDim2.new(1,-4,0,54)
    holder.BackgroundTransparency=1
    holder.Parent=parent

    local label=Instance.new("TextLabel")
    label.Size=UDim2.new(0.68,0,0,18)
    label.Position=UDim2.new(0,2,0,0)
    label.BackgroundTransparency=1
    label.Text=text
    label.TextColor3=Theme.Text
    label.Font=UIFont.Body
    label.TextSize=12
    label.TextXAlignment=Enum.TextXAlignment.Left
    label.Parent=holder

    local box=Instance.new("TextBox")
    box.Size=UDim2.new(0.28,0,0,18)
    box.Position=UDim2.new(0.72,0,0,0)
    box.BackgroundColor3=Theme.InputBg
    box.BackgroundTransparency=UITransparency.Input
    box.BorderSizePixel=0
    box.Text=tostring(default)..(suffix or "")
    box.TextColor3=Theme.Text
    box.Font=UIFont.Button
    box.TextSize=10
    box.ClearTextOnFocus=false
    box.Parent=holder
    corner(box,5)
    stroke(box,Theme.Stroke,1,0.55)

    local bar=Instance.new("Frame")
    bar.Size=UDim2.new(1,-8,0,4)
    bar.Position=UDim2.new(0,2,0,32)
    bar.BackgroundColor3=Theme.SliderBg or Color3.fromRGB(46,52,63)
    bar.BackgroundTransparency=0.25
    bar.BorderSizePixel=0
    bar.Parent=holder
    corner(bar,6)

    local fill=Instance.new("Frame")
    fill.Size=UDim2.new(math.clamp((default-min)/(max-min),0,1),0,1,0)
    fill.BackgroundColor3=Theme.SliderFill or Theme.Accent or Color3.fromRGB(74,140,255)
    fill.BackgroundTransparency=0
    fill.BorderSizePixel=0
    fill.Parent=bar
    corner(fill,6)

    local knob=Instance.new("Frame")
    knob.Name="WhiteSliderKnob"
    knob.Size=UDim2.new(0,12,0,12)
    knob.AnchorPoint=Vector2.new(0.5,0.5)
    knob.Position=UDim2.new(math.clamp((default-min)/(max-min),0,1),0,0.5,0)
    knob.BackgroundColor3=Color3.fromRGB(255,255,255)
    knob.BorderSizePixel=0
    knob.ZIndex=2
    knob.Parent=bar
    corner(knob,20)

    local dragging=false
    local function updateValue(val,skipBox)
        val=math.clamp(val,min,max)
        val=math.floor(val*100+0.5)/100
        local rel=(val-min)/(max-min)
        fill.Size=UDim2.new(rel,0,1,0)
        knob.Position=UDim2.new(rel,0,0.5,0)
        if not skipBox then box.Text=tostring(val)..(suffix or "") end
        if callback then callback(val) end
    end

    bar.InputBegan:Connect(function(i)
        if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then
            dragging=true
            local rel=math.clamp((i.Position.X-bar.AbsolutePosition.X)/bar.AbsoluteSize.X,0,1)
            updateValue(min+(max-min)*rel)
        end
    end)
    UIS.InputEnded:Connect(function(i)
        if i.UserInputType==Enum.UserInputType.MouseButton1 or i.UserInputType==Enum.UserInputType.Touch then dragging=false end
    end)
    UIS.InputChanged:Connect(function(i)
        if dragging and (i.UserInputType==Enum.UserInputType.MouseMovement or i.UserInputType==Enum.UserInputType.Touch) then
            local rel=math.clamp((i.Position.X-bar.AbsolutePosition.X)/bar.AbsoluteSize.X,0,1)
            updateValue(min+(max-min)*rel)
        end
    end)
    box.FocusLost:Connect(function()
        local raw=box.Text:gsub("[^%d%.]","")
        local num=tonumber(raw)
        if num then updateValue(num) else box.Text=tostring(default)..(suffix or "") end
    end)
    return holder
end

function makeMainButton(parent,text,callback,color) local b=Instance.new("TextButton"); b.Size=UDim2.new(1,-4,0,30); b.BackgroundColor3=color or Theme.Row; b.BackgroundTransparency=UITransparency.MainButton; b.Text=text; b.TextColor3=Theme.Text; b.Font=Enum.Font.GothamBold; b.TextSize=11; b.AutoButtonColor=false; b.Parent=parent; corner(b,6); stroke(b,Theme.Stroke,1,0.55)
    b.MouseEnter:Connect(function() tw(b,{BackgroundColor3=color or Theme.RowHover},0.12) end); b.MouseLeave:Connect(function() tw(b,{BackgroundColor3=color or Theme.Row},0.12) end)
    b.MouseButton1Click:Connect(function() if callback then callback() end end); return b end

function makeMainToggle(parent,text,enabled,callback)
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,32); row.BackgroundTransparency=1; row.Parent=parent
    local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-34,1,0); l.Position=UDim2.new(0,2,0,0); l.BackgroundTransparency=1; l.Text=text; l.TextColor3=Theme.Text; l.Font=UIFont.Label; l.TextSize=12; l.TextXAlignment=Enum.TextXAlignment.Left; l.TextTruncate=Enum.TextTruncate.AtEnd; l.Parent=row
    -- Nexus checkbox style v2: small, sharp, outline when off / flat blue when on
    local toggle=Instance.new("TextButton"); toggle.Size=UDim2.new(0,15,0,15); toggle.Position=UDim2.new(1,-21,0.5,-8); toggle.BackgroundTransparency=enabled and 0 or 0.55; toggle.BackgroundColor3=enabled and (Theme.ToggleOn or Theme.Accent) or Theme.ToggleOff; toggle.Text=""; toggle.AutoButtonColor=false; toggle.Parent=row; corner(toggle,3)
    stroke(toggle,enabled and (Theme.ToggleOn or Theme.Accent) or Theme.Stroke,1,0)
    local tStroke=toggle:FindFirstChildOfClass("UIStroke")
    local check=Instance.new("TextLabel"); check.Size=UDim2.new(1,0,1,0); check.Position=UDim2.new(0,0,0,-1); check.BackgroundTransparency=1; check.Text="✓"; check.TextColor3=Theme.ToggleKnobOn or Color3.new(1,1,1); check.Font=Enum.Font.GothamBold; check.TextSize=10; check.Visible=enabled and true or false; check.Parent=toggle
    -- bigger invisible hitbox so the small box stays easy to click
    local hit=Instance.new("TextButton"); hit.Size=UDim2.new(0,30,0,30); hit.Position=UDim2.new(1,-28,0.5,-15); hit.BackgroundTransparency=1; hit.Text=""; hit.Parent=row
    local state=enabled
    local function setState(ns,fire)
        state=ns
        tw(toggle,{BackgroundColor3=state and (Theme.ToggleOn or Theme.Accent) or Theme.ToggleOff, BackgroundTransparency=state and 0 or 0.55},0.1)
        check.Visible=state and true or false
        if tStroke then tStroke.Color=state and (Theme.ToggleOn or Theme.Accent) or Theme.Stroke end
        if fire~=false and callback then callback(state) end
    end
    hit.MouseButton1Click:Connect(function() setState(not state,true) end)
    toggle.MouseButton1Click:Connect(function() setState(not state,true) end)
    BoundToggles[text]=function(ns,fire) if typeof(ns)=="boolean" then setState(ns,fire) else setState(not state,true) end end; return BoundToggles[text]
end

function makeMainTextBox(parent,text,default,placeholder,callback)
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,36); row.BackgroundTransparency=1; row.Parent=parent
    local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-100,1,0); l.Position=UDim2.new(0,2,0,0); l.BackgroundTransparency=1; l.Text=text; l.TextColor3=Theme.Text; l.Font=Enum.Font.GothamMedium; l.TextSize=12; l.TextXAlignment=Enum.TextXAlignment.Left; l.TextTruncate=Enum.TextTruncate.AtEnd; l.Parent=row
    local box=Instance.new("TextBox"); box.Size=UDim2.new(0,84,0,24); box.Position=UDim2.new(1,-90,0.5,-12); box.BackgroundColor3=Theme.InputBg; box.BackgroundTransparency=UITransparency.Input; box.BorderSizePixel=0; box.Text=default or ""; box.PlaceholderText=placeholder or ""; box.TextColor3=Theme.Text; box.Font=Enum.Font.GothamMedium; box.TextSize=10; box.ClearTextOnFocus=false; box.Parent=row; corner(box,6); stroke(box,Theme.Stroke,1,0.65)
    box.FocusLost:Connect(function()
        local raw = box.Text:gsub("%s", "")
        if callback then callback(raw) end
    end)
    return box
end

function makeKeybindRow(parent,nameText,displayText)
    displayText = displayText or nameText
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,36); row.BackgroundTransparency=1; row.Parent=parent
    local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-88,1,0); l.Position=UDim2.new(0,2,0,0); l.BackgroundTransparency=1; l.Text=displayText; l.TextColor3=Theme.Text; l.Font=Enum.Font.GothamMedium; l.TextSize=12; l.TextXAlignment=Enum.TextXAlignment.Left; l.Parent=row
    local x=Instance.new("TextButton"); x.Name="WhiteTextBtn"; x.Size=UDim2.new(0,22,0,20); x.Position=UDim2.new(1,-74,0.5,-10); x.BackgroundColor3=Theme.Red; x.Text="X"; x.TextColor3=Color3.new(1,1,1); x.Font=Enum.Font.GothamBold; x.TextSize=10; x.Parent=row; corner(x,5)
    local key=Instance.new("TextButton"); key.Name="WhiteTextBtn"; key.Size=UDim2.new(0,50,0,20); key.Position=UDim2.new(1,-50,0.5,-10); key.BackgroundColor3=Theme.Row; key.BackgroundTransparency=UITransparency.Input; key.Text=Keybinds[nameText] or "NONE"; key.TextColor3=Color3.new(1,1,1); key.Font=Enum.Font.GothamBold; key.TextSize=9; key.Parent=row; corner(key,5); stroke(key,Theme.Stroke,1,0.45)
    x.MouseButton1Click:Connect(function() Keybinds[nameText]="NONE"; Config.keybinds[nameText]="NONE"; saveConfig(); key.Text="NONE"; if updateMovementPanelLabels then updateMovementPanelLabels() end end)
    key.MouseButton1Click:Connect(function() key.Text="..."
        local con; con=UIS.InputBegan:Connect(function(input,gp) if gp then return end; if input.UserInputType==Enum.UserInputType.Keyboard then Keybinds[nameText]=input.KeyCode.Name; Config.keybinds[nameText]=input.KeyCode.Name; saveConfig(); key.Text=input.KeyCode.Name; if nameText=="Open Menu" then UI.OpenMenuKey=input.KeyCode end; con:Disconnect(); if updateMovementPanelLabels then updateMovementPanelLabels() end end end)
    end)
end

-- CREATE PANELS
main,mainBody=makeMainPanel("Sxe Remasterd",UDim2.new(0,520,0,440),UDim2.new(0.5,-260,0.5,-220))
if Config.AutoCloseOnExec then main.Visible = false end
panels["Invisible Steal Panel"],panels["InvisStealBody"]=makeQuickPanel("Sxe Remasterd\nInvisible Steal",UDim2.new(0,230,0,375),UDim2.new(0,80,0.5,-220))
panels["InvisStealBody"].ScrollBarThickness = 2
panels["InvisStealBody"].ScrollingEnabled = true
do
    local invisLay = panels["InvisStealBody"]:FindFirstChildOfClass("UIListLayout")
    if invisLay then invisLay.Padding = UDim.new(0, 10) end
end
panels["Admin Command Panel"],panels["AdminBody"]=makeQuickPanel("Sxe Remasterd\nAdmin Command Panel",UDim2.new(0,225,0,240),UDim2.new(0.5,85,1,-340))
panels["Command Cooldowns"],panels["CooldownBody"]=makeQuickPanel("Sxe Remasterd\nCommand Cooldowns",UDim2.new(0,210,0,315),UDim2.new(0.5,245,1,-390))
panels["Actions"],panels["ActionsBody"]=makeQuickPanel("Sxe Remasterd\nActions",UDim2.new(0,230,0,340),UDim2.new(0.5,505,1,-415))
panels["Steal Panel"],panels["StealBody"]=makeQuickPanel("Sxe Remasterd\nSteal Panel",UDim2.new(0,235,0,335),UDim2.new(1,-300,1,-420))
panels["Steal Target"],panels["TargetBody"]=makeQuickPanel("Sxe Remasterd\nSteal Target",UDim2.new(0,320,0,530),UDim2.new(1,-330,0,85))
-- Funcao compartilhada: acha o melhor Model 3D de verdade (nao o hitbox invisivel)
-- pra um pet, usada tanto no viewer grande quanto nos mini-icones da lista Steal Target.
function findBestAnimalModel(petData)
    if not petData then return nil end
    local sourceModel = nil
    pcall(function()
        local ad = petData.animalData or petData
        local plots = Workspace:FindFirstChild("Plots")
        local plot = plots and plots:FindFirstChild(ad.plot)
        if not plot then return end
        local podiums = plot:FindFirstChild("AnimalPodiums")
        local podium = podiums and podiums:FindFirstChild(ad.slot)
        local spawnPart = podium and podium:FindFirstChild("Base")
        if not spawnPart then return end
        local spawn = spawnPart:FindFirstChild("Spawn") or spawnPart:FindFirstChildWhichIsA("BasePart") or spawnPart
        local spawnPos = spawn.Position

        -- Tenta primeiro dentro do podium (filhos diretos do Spawn)
        for _, child in ipairs(spawn:GetChildren()) do
            if child:IsA("Model") then
                local root = child:FindFirstChild("HumanoidRootPart") or child.PrimaryPart or child:FindFirstChildWhichIsA("BasePart")
                if root and (root.Position - spawnPos).Magnitude < 6 then
                    local hasAnim = child:FindFirstChildWhichIsA("AnimationController") or child:FindFirstChild("Humanoid")
                    local visible = 0
                    for _, d in ipairs(child:GetDescendants()) do
                        if d:IsA("MeshPart") and d.Transparency < 1 then visible += 1 end
                    end
                    if visible > 0 or hasAnim then
                        sourceModel = child
                        return
                    end
                end
            end
        end

        -- Fallback: scan no plot com filtro melhor
        local bestScore, bestModel = -1, nil
        for _, obj in ipairs(plot:GetDescendants()) do
            if not obj:IsA("Model") or obj.Name=="Base" then continue end
            if obj:FindFirstChild("Base") or obj:FindFirstChild("Spawn") then continue end
            local root = obj:FindFirstChild("HumanoidRootPart") or obj.PrimaryPart or obj:FindFirstChildWhichIsA("BasePart")
            if not root then continue end
            local dist = (root.Position - spawnPos).Magnitude
            if dist > 12 then continue end
            local score = -dist
            local iname = ad.animalIndex or ad.index or ""
            if iname~="" and (obj.Name==iname or obj.Name:find(iname,1,true)) then score+=500 end
            if (petData.petName or petData.name or "")~="" and obj.Name:find(petData.petName or petData.name or "",1,true) then score+=200 end
            local hasAnim = obj:FindFirstChildWhichIsA("AnimationController") or obj:FindFirstChild("Humanoid")
            if hasAnim then score+=150 end
            local visibleMeshParts = 0
            for _, d in ipairs(obj:GetDescendants()) do
                if d:IsA("MeshPart") and d.Transparency < 1 then visibleMeshParts += 1 end
            end
            if visibleMeshParts > 0 then
                score += math.min(visibleMeshParts * 10, 100)
            else
                if not hasAnim then score -= 300 end
            end
            if root:IsA("BasePart") and root.Transparency >= 1 and visibleMeshParts == 0 then score -= 300 end
            if score>bestScore then bestScore=score; bestModel=obj end
        end
        sourceModel = bestModel
    end)
    return sourceModel
end

actionSettingsPanel,actionSettingsBody=makeQuickPanel("Sxe Remasterd\nAction Settings",UDim2.new(0,230,0,370),UDim2.new(0.5,745,1,-440))
actionSettingsPanel.Visible=false
tpSpeedSettingsPanel,tpSpeedSettingsBody=makeQuickPanel("Sxe Remasterd\nTP & Clone Settings",UDim2.new(0,235,0,325),UDim2.new(0.5,745,1,-440))
tpSpeedSettingsPanel.Visible=false
for _,pair in ipairs({{"Sxe Remasterd",main},{"Sxe Remasterd\nInvisible Steal",panels["Invisible Steal Panel"]},
    {"Sxe Remasterd\nAdmin Command Panel",panels["Admin Command Panel"]},{"Sxe Remasterd\nCommand Cooldowns",panels["Command Cooldowns"]},
    {"Sxe Remasterd\nActions",panels["Actions"]},{"Sxe Remasterd\nSteal Panel",panels["Steal Panel"]},{"Sxe Remasterd\nSteal Target",panels["Steal Target"]},
    {"Sxe Remasterd\nAction Settings",actionSettingsPanel},{"Sxe Remasterd\nTP & Clone Settings",tpSpeedSettingsPanel}}) do applySavedPosition(pair[1],pair[2]) end


-- ============================================================
-- GRIEF DETECTOR (Nexus): admin panel commands + grief gears.
-- Server-signal based (attributes + debris in char) -> no false
-- positives: own Tools/Accessories ignored, no StateChanged.
-- ============================================================
task.spawn(function() -- own function scope: keeps locals off the main
    -- chunk's 200-register cap
    local GD_LOG = {}
    local gdAvatars = {}
    local gdExpanded = false
    local gdLastFire = {}
    local GD_COOLDOWN = 8

    local GD_ADMIN = { balloon=true, inverse=true, inversed=true, jail=true, jumpscare=true, morph=true, nightvision=true, rocket=true, tiny=true, ragdoll=true }
    -- full gear list from the wiki (stealabrainrot.fandom.com/wiki/Gears).
    -- Specific slap tiers FIRST so they win over the generic "slap" match.
    local GD_GEARS = {
        -- every slap tier
        {kw="dark matter slap",name="Dark Matter Slap"},
        {kw="nuclear slap",name="Nuclear Slap"},
        {kw="glitched slap",name="Glitched Slap"},
        {kw="galaxy slap",name="Galaxy Slap"},
        {kw="flame slap",name="Flame Slap"},
        {kw="emerald slap",name="Emerald Slap"},
        {kw="diamond slap",name="Diamond Slap"},
        {kw="ruby slap",name="Ruby Slap"},
        {kw="gold slap",name="Gold Slap"},
        {kw="iron slap",name="Iron Slap"},
        {kw="gummy slap",name="Gummy Slap"},
        {kw="splatter slap",name="Splatter Slap"},
        {kw="slap",name="Slap"},
        -- offensive gears
        {kw="airstrike",name="Airstrike"},
        {kw="air strike",name="Airstrike"},
        {kw="trap",name="Trap"},
        {kw="rage table",name="Rage Table"},
        {kw="taser",name="Taser"},
        {kw="laser cape",name="Laser Cape"},
        {kw="paintball",name="Paintball Gun"},
        {kw="megaphone",name="Megaphone"},
        {kw="gummy",name="Gummy Slap"},
        {kw="subspace",name="Subspace Mine"},
        {kw="mine",name="Subspace Mine"},
        {kw="attack doge",name="Attack Doge"},
        {kw="doge",name="Attack Doge"},
        {kw="medusa",name="Medusa's Head"},
        {kw="web slinger",name="Web Slinger"},{kw="webslinger",name="Web Slinger"},
        {kw="boogie",name="Boogie Bomb"},
        {kw="body swap",name="Body Swap Potion"},{kw="bodyswap",name="Body Swap Potion"},
        {kw="bee launcher",name="Bee Launcher"},{kw="beehive",name="Beehive"},
        {kw="magnet",name="Magnet"},
        {kw="heatseeker",name="Heatseeker"},
        {kw="sentry",name="All-Seeing Sentry"},
        {kw="rainbowrath",name="Rainbowrath Sword"},
    }
    local function gdMatchGear(s)
        s=tostring(s):lower()
        for _,g in ipairs(GD_GEARS) do if s:find(g.kw,1,true) then return g.name end end
        return nil
    end

    -- EFFECT keywords: what griefs actually LEAVE on you (server attributes
    -- and effect objects in your character). Boogie/body swap/gummy... give
    -- no "gear-named" debris, so they need their own matching table.
    local GD_EFFECTS = {
        {kw="boogie",name="Boogie Bomb"},{kw="dancing",name="Boogie Bomb"},{kw="dance",name="Boogie Bomb"},
        {kw="bodyswap",name="Body Swap Potion"},{kw="swapped",name="Body Swap Potion"},{kw="swap",name="Body Swap Potion"},
        {kw="gummy",name="Gummy Slap"},{kw="splat",name="Splatter Slap"},
        {kw="webbed",name="Web Slinger"},{kw="web",name="Web Slinger"},
        {kw="beehive",name="Beehive"},{kw="bee",name="Bee (inverted)"},
        {kw="stunned",name="Taser / Stun"},{kw="stun",name="Taser / Stun"},{kw="tased",name="Taser"},
        {kw="frozen",name="Freeze"},{kw="freeze",name="Freeze"},
        {kw="paint",name="Paintball Gun"},{kw="magnet",name="Magnet"},
        {kw="medusa",name="Medusa's Head"},{kw="petrif",name="Medusa's Head"},{kw="stoned",name="Medusa's Head"},
        {kw="subspace",name="Subspace Mine"},{kw="doge",name="Attack Doge"},
        {kw="lasered",name="Laser Cape"},{kw="rage",name="Rage Table"},
        {kw="invert",name="Inverted Controls"},
    }
    local function gdMatchEffect(s)
        s=tostring(s):lower()
        for _,g in ipairs(GD_EFFECTS) do if s:find(g.kw,1,true) then return g.name end end
        return nil
    end

    local gdPanel=Instance.new("Frame")
    gdPanel.Name="GriefDetector"
    gdPanel.Size=UDim2.new(0,250,0,132)
    gdPanel.Position=UDim2.new(1,-270,0,500)
    gdPanel.BackgroundColor3=Theme.Background
    gdPanel.BackgroundTransparency=UITransparency.Panel
    gdPanel.BorderSizePixel=0
    gdPanel.ClipsDescendants=true
    gdPanel.Parent=gui
    corner(gdPanel,10); addOutline(gdPanel)

    local gdHeader=Instance.new("Frame"); gdHeader.Size=UDim2.new(1,0,0,28); gdHeader.BackgroundColor3=Theme.Panel; gdHeader.BackgroundTransparency=0.1; gdHeader.BorderSizePixel=0; gdHeader.Parent=gdPanel; corner(gdHeader,10)
    local gdTitle=Instance.new("TextLabel"); gdTitle.Size=UDim2.new(1,-16,1,0); gdTitle.Position=UDim2.new(0,10,0,0); gdTitle.BackgroundTransparency=1; gdTitle.Font=Enum.Font.GothamBlack; gdTitle.TextSize=12; gdTitle.TextColor3=Theme.Text; gdTitle.TextXAlignment=Enum.TextXAlignment.Left; gdTitle.Text="GRIEF DETECTOR"; gdTitle.Parent=gdHeader

    local gdList=Instance.new("ScrollingFrame")
    gdList.Position=UDim2.new(0,8,0,32)
    gdList.Size=UDim2.new(1,-16,0,56)
    gdList.BackgroundTransparency=1; gdList.BorderSizePixel=0
    gdList.ScrollBarThickness=2; gdList.ScrollBarImageColor3=Theme.Stroke
    gdList.CanvasSize=UDim2.new(0,0,0,0); gdList.ScrollingEnabled=false
    gdList.Parent=gdPanel
    local gdLay=Instance.new("UIListLayout"); gdLay.Padding=UDim.new(0,6); gdLay.SortOrder=Enum.SortOrder.LayoutOrder; gdLay.Parent=gdList
    gdLay:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function() gdList.CanvasSize=UDim2.new(0,0,0,gdLay.AbsoluteContentSize.Y+4) end)

    local function gdResize()
        -- fixed compact height; the list scrolls through the full history
        gdList.Size=UDim2.new(1,-16,0,68)
        gdList.ScrollingEnabled=true
        gdPanel.Size=UDim2.new(0,250,0,110)
    end

    local function gdRebuild()
        for _,c in ipairs(gdList:GetChildren()) do if not c:IsA("UIListLayout") then c:Destroy() end end
        if #GD_LOG==0 then
            local e=Instance.new("TextLabel"); e.Size=UDim2.new(1,0,0,50); e.BackgroundTransparency=1; e.Text="No griefs yet."; e.TextColor3=Theme.Dim; e.Font=Enum.Font.GothamMedium; e.TextSize=13; e.Parent=gdList
            return
        end
        for i=1,math.min(#GD_LOG,20) do
            local g=GD_LOG[i]
            local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,64); row.BackgroundColor3=Theme.Row; row.BackgroundTransparency=UITransparency.Row; row.BorderSizePixel=0; row.LayoutOrder=i; row.Parent=gdList; corner(row,8)
            local av=Instance.new("ImageLabel"); av.Size=UDim2.fromOffset(40,40); av.Position=UDim2.new(0,6,0.5,-20); av.BackgroundColor3=Theme.Panel; av.BackgroundTransparency=0.2; av.BorderSizePixel=0; av.Parent=row; corner(av,8)
            if g.userId then
                if gdAvatars[g.userId] then av.Image=gdAvatars[g.userId]
                else task.spawn(function() pcall(function()
                    local img=Players:GetUserThumbnailAsync(g.userId,Enum.ThumbnailType.HeadShot,Enum.ThumbnailSize.Size48x48)
                    gdAvatars[g.userId]=img; if av.Parent then av.Image=img end
                end) end) end
            end
            local nm=Instance.new("TextLabel"); nm.Size=UDim2.new(1,-56,0,13); nm.Position=UDim2.new(0,52,0,4); nm.BackgroundTransparency=1; nm.Text=g.who; nm.TextColor3=Theme.Accent or Color3.fromRGB(74,140,255); nm.Font=Enum.Font.GothamBold; nm.TextSize=12; nm.TextXAlignment=Enum.TextXAlignment.Left; nm.TextTruncate=Enum.TextTruncate.AtEnd; nm.Parent=row
            -- line 2: gear / admin panel command that griefed
            local gt=Instance.new("TextLabel"); gt.Size=UDim2.new(1,-56,0,11); gt.Position=UDim2.new(0,52,0,18); gt.BackgroundTransparency=1; gt.Text=g.grief; gt.TextColor3=Theme.Dim; gt.Font=Enum.Font.GothamMedium; gt.TextSize=10; gt.TextXAlignment=Enum.TextXAlignment.Left; gt.TextTruncate=Enum.TextTruncate.AtEnd; gt.Parent=row
            -- line 3: the brainrot he was carrying [rarity]
            local br=Instance.new("TextLabel"); br.Size=UDim2.new(1,-56,0,12); br.Position=UDim2.new(0,52,0,31); br.BackgroundTransparency=1
            br.Text=g.carried or ((g.tool and (g.tool.."  —  ") or "")..(g.dist and (g.dist.."m") or ""))
            br.TextColor3=Theme.Text; br.Font=Enum.Font.GothamBold; br.TextSize=10; br.TextXAlignment=Enum.TextXAlignment.Left; br.TextTruncate=Enum.TextTruncate.AtEnd; br.Parent=row
            -- line 4: its generation per second
            local gn=Instance.new("TextLabel"); gn.Size=UDim2.new(1,-56,0,12); gn.Position=UDim2.new(0,52,0,45); gn.BackgroundTransparency=1
            gn.Text=g.carriedGen or ""
            gn.TextColor3=Color3.fromRGB(100,220,130); gn.Font=Enum.Font.GothamBold; gn.TextSize=10; gn.TextXAlignment=Enum.TextXAlignment.Left; gn.TextTruncate=Enum.TextTruncate.AtEnd; gn.Parent=row
        end
    end

    makeDraggable(gdPanel,gdHeader,"Mynxx\nGrief Detector")
    applySavedPosition("Mynxx\nGrief Detector",gdPanel)
    panels["Grief Detector"]=gdPanel
    gdResize(); gdRebuild()

    -- attribution: nearest player HOLDING a grief gear wins; else nearest
    -- player, but without naming his item (mobility tools can't grief)
    local function gdNearestGriefer()
        local myHrp=player.Character and player.Character:FindFirstChild("HumanoidRootPart")
        if not myHrp then return nil end
        local gp,gpD,gpT=nil,math.huge,nil
        local np,npD=nil,math.huge
        for _,p in ipairs(Players:GetPlayers()) do
            if p~=player and p.Character then
                local hrp=p.Character:FindFirstChild("HumanoidRootPart")
                if hrp then
                    local d=(hrp.Position-myHrp.Position).Magnitude
                    if d<150 then
                        if d<npD then npD,np=d,p end
                        local t=p.Character:FindFirstChildOfClass("Tool")
                        local gname=t and gdMatchGear(t.Name)
                        if gname and d<gpD then gpD,gp,gpT=d,p,t.Name end
                    end
                end
            end
        end
        if gp then return gp,gpT,math.floor(gpD) end
        if np then return np,nil,math.floor(npD) end
        return nil
    end

    -- STRICT slap attribution: a slap only lands at melee range, so the real
    -- slapper is the gear holder within 20 studs -- melee gears (slap tiers /
    -- rainbowrath / rage) preferred over any other gear. Fixes wrong
    -- person/item on ragdoll.
    local function gdSlapper()
        local myHrp=player.Character and player.Character:FindFirstChild("HumanoidRootPart")
        if not myHrp then return nil end
        local best,bestD,bestTool=nil,math.huge,nil
        local melee,meleeD,meleeTool=nil,math.huge,nil
        for _,p in ipairs(Players:GetPlayers()) do
            if p~=player and p.Character then
                local hrp=p.Character:FindFirstChild("HumanoidRootPart")
                local t=p.Character:FindFirstChildOfClass("Tool")
                if hrp and t and gdMatchGear(t.Name) then
                    local d=(hrp.Position-myHrp.Position).Magnitude
                    if d<=20 then
                        local tn=t.Name:lower()
                        if tn:find("slap",1,true) or tn:find("rainbowrath",1,true) or tn:find("rage",1,true) then
                            if d<meleeD then meleeD,melee,meleeTool=d,p,t.Name end
                        end
                        if d<bestD then bestD,best,bestTool=d,p,t.Name end
                    end
                end
            end
        end
        if melee then return melee,meleeTool,math.floor(meleeD) end
        if best then return best,bestTool,math.floor(bestD) end
        return nil
    end

    -- MY CARRIED BRAINROT tracker: while I'm carrying, remember what it is;
    -- a grief usually makes me drop it (attr flips off instantly), so the
    -- snapshot is what gets shown when the grief lands.
    local gdMyCarry=nil -- {txt=name [rarity], gen="$X/s", t=os.clock()}
    local gdBoogie=nil -- last seen nearby Boogie Bomb holder {plr,dist,t}
    local gdAirstrike=nil -- last seen nearby Airstrike holder {plr,dist,t}
    local gdLastJumpT=0 -- last big position jump (body-swap signature)
    local function gdFmtGen(n)
        n=tonumber(n)
        if not n or n<=0 then return nil end
        if n>=1e9 then return string.format("$%.2fB/s",n/1e9) end
        if n>=1e6 then return string.format("$%.2fM/s",n/1e6) end
        if n>=1e3 then return string.format("$%.1fk/s",n/1e3) end
        return "$"..math.floor(n).."/s"
    end
    task.spawn(function()
        while true do
            task.wait(0.25)
            -- track nearby Boogie Bomb / Airstrike HOLDERS (same info as the
            -- held-tool ESP): used to blame them when the effect lands even
            -- if they already unequipped the gear by then
            pcall(function()
                local myHrp=player.Character and player.Character:FindFirstChild("HumanoidRootPart")
                if not myHrp then return end
                for _,p in ipairs(Players:GetPlayers()) do
                    if p~=player and p.Character then
                        local t=p.Character:FindFirstChildOfClass("Tool")
                        if t then
                            local tn=t.Name:lower()
                            local hrp=p.Character:FindFirstChild("HumanoidRootPart")
                            local d=hrp and (hrp.Position-myHrp.Position).Magnitude
                            if d and d<=100 then
                                if tn:find("boogie",1,true) and d<=80 then
                                    gdBoogie={plr=p,dist=math.floor(d),t=os.clock()}
                                elseif tn:find("airstrike",1,true) or tn:find("air strike",1,true) then
                                    gdAirstrike={plr=p,dist=math.floor(d),t=os.clock()}
                                end
                            end
                        end
                    end
                end
            end)
            if player:GetAttribute("Stealing") then
                local nm,mut,genTxt=nil,nil,nil
                -- 1) our own steal engine knows exactly what we grabbed
                pcall(function()
                    local st=_G.Mynxx_StealStatus
                    if st and st.target and st.target.name then
                        nm=st.target.name
                        genTxt=gdFmtGen(st.target.mps or st.target.value)
                    end
                end)
                pcall(function()
                    local sel=SharedState.SelectedPetData
                    if sel and (sel.petName or sel.name) then
                        if not nm then nm=sel.petName or sel.name end
                        if not mut then mut=sel.mutation end
                        if not genTxt then
                            if sel.mpsText and tostring(sel.mpsText)~="" then
                                local g=tostring(sel.mpsText):gsub("^%$","")
                                if not g:lower():find("/s") then g=g.."/s" end
                                genTxt="$"..g
                            else
                                genTxt=gdFmtGen(sel.mpsValue)
                            end
                        end
                    end
                end)
                -- 2) fallback: animal model attached to MY character
                if not nm then
                    pcall(function()
                        if _MynxxAnimalNames and player.Character then
                            for _,m in ipairs(player.Character:GetDescendants()) do
                                if m:IsA("Model") or m:IsA("MeshPart") then
                                    local hit=_MynxxAnimalNames[m.Name:lower()]
                                    if hit then nm=hit; break end
                                end
                            end
                        end
                    end)
                end
                if nm then
                    gdMyCarry={
                        txt=tostring(nm)..((mut and mut~="" and tostring(mut):lower()~="normal") and (" ["..tostring(mut).."]") or ""),
                        gen=genTxt, t=os.clock(),
                    }
                end
            end
        end
    end)

    local function gdReport(griefName,fPlr,fTool,fDist)
        local now=os.clock()
        -- self-inflicted admin command (e.g. Ragdoll Self action) -> no grief
        if (now-(_G.__MynxxSelfCmdT or -100))<3 then return end
        if gdLastFire[griefName] and (now-gdLastFire[griefName])<GD_COOLDOWN then return end
        gdLastFire[griefName]=now
        local plr,tool,dist
        if fPlr then plr,tool,dist=fPlr,fTool,fDist
        else plr,tool,dist=gdNearestGriefer() end
        -- grief label: show the actual GEAR he's holding (or the admin panel
        -- command) instead of the generic "Ragdoll / Slap"
        local griefLabel=griefName
        if griefName=="Ragdoll / Slap" and tool then
            local g=gdMatchGear(tool)
            if g then griefLabel=g end
        end
        -- 1st priority: the brainrot I WAS CARRYING when the grief landed
        -- (saved by the tracker above; a grief drops it instantly so the
        -- live attribute is already gone by now)
        local carried,carriedGen=nil,nil
        if gdMyCarry and (now-gdMyCarry.t)<10 then
            carried=gdMyCarry.txt
            carriedGen=gdMyCarry.gen
        end
        -- 2nd: the brainrot in the GRIEFER's hands
        if not carried and plr then
            local petName
            pcall(function()
                local _,n=getStealingInfo(plr)
                if n and n~="Brainrot" then petName=n end
            end)
            if not petName then
                pcall(function()
                    if _MynxxAnimalNames and plr.Character then
                        for _,m in ipairs(plr.Character:GetDescendants()) do
                            if m:IsA("Model") or m:IsA("MeshPart") then
                                local hit=_MynxxAnimalNames[m.Name:lower()]
                                if hit then petName=hit; break end
                            end
                        end
                    end
                end)
            end
            if petName then
                local mut,gen=nil,nil
                pcall(function()
                    for _,a in ipairs(SharedState.AllAnimalsCache or {}) do
                        if a and a.name and a.name:lower()==tostring(petName):lower() then
                            mut=a.mutation; gen=a.genText; break
                        end
                    end
                end)
                carried=tostring(petName)..((mut and mut~="" and tostring(mut):lower()~="normal") and (" ["..tostring(mut).."]") or "")
                if gen and tostring(gen)~="" then
                    gen=tostring(gen):gsub("^%$","")
                    if not gen:lower():find("/s") then gen=gen.."/s" end
                    carriedGen="$"..gen
                end
            end
        end
        table.insert(GD_LOG,1,{
            who=plr and (plr.DisplayName.." (@"..plr.Name..")") or "Unknown",
            userId=plr and plr.UserId or nil,
            grief=griefLabel, tool=tool, dist=dist, carried=carried, carriedGen=carriedGen, t=now,
        })
        while #GD_LOG>20 do table.remove(GD_LOG) end
        gdRebuild()
        pcall(function()
            local o=gdPanel:FindFirstChildOfClass("UIStroke")
            if o then
                o.Color=Theme.Accent or Color3.fromRGB(74,140,255); o.Transparency=0
                task.delay(1,function() o.Color=Theme.Stroke; o.Transparency=0.35 end)
            end
        end)
    end

    -- SIGNAL 1: server attributes = exact truth, no guessing
    local function gdOnAttr(inst,attr)
        local v=inst:GetAttribute(attr)
        if v==false or v==nil then return end
        local n=tostring(attr):lower()
        if n=="stealing" or n=="stealingindex" then return end
        if n:sub(1,7)=="ragdoll" then
            -- ragdoll attr carries the end time: expired = not a live grief
            if type(v)=="number" and v<=Workspace:GetServerTimeNow() then return end
            -- strict attribution: melee-range slapper first, then airstrike
            local sp,st,sd=gdSlapper()
            if sp then
                gdReport("Ragdoll / Slap",sp,st,sd)
            elseif gdAirstrike and (os.clock()-gdAirstrike.t)<8 then
                gdReport("Airstrike",gdAirstrike.plr,"Airstrike",gdAirstrike.dist)
            else
                gdReport("Ragdoll / Slap")
            end
            return
        end
        for cmd in pairs(GD_ADMIN) do
            if cmd~="ragdoll" and n:sub(1,#cmd)==cmd then
                gdReport(cmd:sub(1,1):upper()..cmd:sub(2).." (Admin Panel)"); return
            end
        end
        -- gear EFFECTS set as server attributes (boogie/swapped/stunned...)
        local eff=gdMatchEffect(n)
        if eff then gdReport(eff) end
    end
    local function gdHookAttrs(inst)
        if not inst then return end
        pcall(function() inst.AttributeChanged:Connect(function(a) gdOnAttr(inst,a) end) end)
    end

    -- SIGNAL 2: grief debris parented into MY character (webs, paint, bees).
    -- Tools/Accessories are OUR OWN equips & cosmetics -> ignored (this was
    -- the old detector's main false-positive source). No StateChanged either
    -- (falling over used to count as a grief).
    local function gdHookChar(char)
        if not char then return end
        gdHookAttrs(char)
        pcall(function()
            char.DescendantAdded:Connect(function(obj)
                if obj:IsA("Tool") or obj:IsA("Accessory") then return end
                -- parts INSIDE an equipped tool are our own gear too
                if obj:FindFirstAncestorOfClass("Tool") then return end
                local nm=obj.Name:lower()
                -- admin panel effects that attach objects to the character
                for _,cmd in ipairs({"balloon","rocket","jail","jumpscare","nightvision"}) do
                    if nm:find(cmd,1,true) then
                        gdReport(cmd:sub(1,1):upper()..cmd:sub(2).." (Admin Panel)")
                        return
                    end
                end
                -- gear-named debris OR effect debris (bees, webs, dance...)
                local gname=gdMatchGear(obj.Name) or gdMatchEffect(obj.Name)
                if gname then gdReport(gname) end
            end)
        end)
        -- INSTANT slap/hit trigger: the physics ragdoll happens client-side
        -- immediately, while the server attribute replicates 1-2s later.
        -- Gated hard (grief gear in hand + <=35 studs) so normal falls or
        -- random trips can NEVER count as a grief.
        local hum=char:FindFirstChildOfClass("Humanoid")
        if hum then
            pcall(function()
                hum.StateChanged:Connect(function(_,new)
                    if new==Enum.HumanoidStateType.Ragdoll
                        or new==Enum.HumanoidStateType.FallingDown
                        or new==Enum.HumanoidStateType.Physics then
                        local p2,tool2,dist2=gdSlapper()
                        if p2 then
                            -- exact slapper + his exact item, forced through
                            gdReport("Ragdoll / Slap",p2,tool2,dist2)
                        elseif gdAirstrike and (os.clock()-gdAirstrike.t)<8 then
                            -- ragdolled with nobody in slap range but someone
                            -- had an Airstrike in hand -> it's the airstrike
                            gdReport("Airstrike",gdAirstrike.plr,"Airstrike",gdAirstrike.dist)
                        end
                    end
                end)
            end)
            -- BOOGIE BOMB: no attribute, no debris -- its signature is the
            -- forced dance ANIMATION played on our humanoid
            pcall(function()
                hum.AnimationPlayed:Connect(function(track)
                    local nm=""
                    pcall(function()
                        nm=((track.Animation and track.Animation.Name) or "").." "..(track.Name or "")
                    end)
                    nm=nm:lower()
                    if nm:find("boogie",1,true) then
                        gdReport("Boogie Bomb")
                    elseif nm:find("dance",1,true) or nm:find("twerk",1,true) then
                        -- generic dance anim: only counts with an armed
                        -- griefer nearby (so OUR OWN /e dance never counts)
                        local p2,tool2,dist2=gdNearestGriefer()
                        if p2 and tool2 and dist2 and dist2<=80 then gdReport("Boogie Bomb") end
                    end
                end)
            end)
            -- GUMMY/TRAP/TASER freeze: our WalkSpeed gets forced to 0
            pcall(function()
                hum:GetPropertyChangedSignal("WalkSpeed"):Connect(function()
                    if hum.WalkSpeed<=0.5 then
                        if Config and Config.WalkSpeedEnabled then return end
                        local p2,tool2,dist2=gdNearestGriefer()
                        if p2 and tool2 and dist2 and dist2<=35 then gdReport("Freeze / Stun") end
                    end
                end)
            end)
        end
        -- BODY SWAP POTION: I get TELEPORTED while CARRYING a brainrot and
        -- the brainrot is TAKEN from me. That triple signature can't happen
        -- by accident: own TPs excluded (_G._isTpMoving), vertical launches
        -- (rocket) excluded, normal delivery has no teleport.
        task.spawn(function()
            local hrp=char:FindFirstChild("HumanoidRootPart") or char:WaitForChild("HumanoidRootPart",5)
            if not hrp then return end
            local lastPos=hrp.Position
            while hrp.Parent and char.Parent do
                RunService.Heartbeat:Wait()
                local p=hrp.Position
                local d=p-lastPos
                lastPos=p
                local horiz=Vector3.new(d.X,0,d.Z).Magnitude
                if horiz>25 and math.abs(d.Y)<horiz+5 and not _G._isTpMoving then
                    gdLastJumpT=os.clock()
                    local wasCarrying=(player:GetAttribute("Stealing")==true)
                        or (gdMyCarry and (os.clock()-gdMyCarry.t)<1.5)
                    if wasCarrying then
                        -- confirm the brainrot was yanked right after the TP
                        task.delay(0.25,function()
                            if player:GetAttribute("Stealing")~=true then
                                gdReport("Body Swap Potion")
                            end
                        end)
                    end
                end
            end
        end)
    end
    gdHookAttrs(player)
    if player.Character then gdHookChar(player.Character) end
    player.CharacterAdded:Connect(function(c) task.wait(0.2); gdHookChar(c) end)

    -- SIGNAL 5: ADMIN PANEL commands. The game writes the active effect on
    -- the OVERHEAD TAG of the victim (the hub already reads it on other
    -- players for the admin rows) -- poll OUR OWN tag for it. Covers
    -- inverse/tiny/morph... that leave no object and no attribute.
    task.spawn(function()
        local FX={"invers","jail","jumpscare","morph","nightvision","rocket","tiny","balloon"}
        local lastFx=nil
        while true do
            task.wait(0.5)
            local fx=nil
            pcall(function()
                local char=player.Character
                if not char then return end
                for _,bb in ipairs(char:GetDescendants()) do
                    if bb:IsA("BillboardGui") and bb.Enabled and not isOurOverheadBillboard(bb) then
                        for _,lbl in ipairs(bb:GetDescendants()) do
                            if lbl:IsA("TextLabel") and lbl.Visible~=false then
                                local t=tostring(lbl.Text):lower()
                                for _,kw in ipairs(FX) do
                                    if t:find(kw,1,true) then fx=kw; return end
                                end
                            end
                        end
                    end
                end
            end)
            if fx and fx~=lastFx then
                lastFx=fx
                local label=(fx=="invers") and "inverse" or fx
                gdReport(label:sub(1,1):upper()..label:sub(2).." (Admin Panel)")
            elseif not fx then
                lastFx=nil
            end
        end
    end)

    -- SIGNAL 4: BOOGIE correlation. The boogie leaves no attribute/debris,
    -- but: someone HOLDING a Boogie Bomb near me (tracked above, like the
    -- held-tool ESP) + I DROP my carried brainrot = boogie grief. Delivering
    -- at my own base, dying, and body-swap TPs are all excluded.
    pcall(function()
        player:GetAttributeChangedSignal("Stealing"):Connect(function()
            if player:GetAttribute("Stealing")==true then return end
            -- I just stopped carrying: was it a grief-drop?
            if not (gdMyCarry and (os.clock()-gdMyCarry.t)<2) then return end
            if (os.clock()-gdLastJumpT)<0.6 then return end -- body swap already handles this
            local hum=player.Character and player.Character:FindFirstChildOfClass("Humanoid")
            if hum and hum.Health<=0 then return end
            local delivered=false
            pcall(function()
                delivered=(_G.__getCurrentBaseOwnerId and _G.__getCurrentBaseOwnerId()==player.UserId) or false
            end)
            if delivered then return end
            if gdBoogie and (os.clock()-gdBoogie.t)<6 then
                gdReport("Boogie Bomb",gdBoogie.plr,"Boogie Bomb",gdBoogie.dist)
            end
        end)
    end)

    -- SIGNAL 3: PLACED/THROWN gears. The bomb/trap model spawns in the WORLD
    -- the instant it's placed -- catch it by name within 60 studs of us and
    -- blame whoever is closest. (Objects inside player characters are just
    -- someone EQUIPPING the gear -> ignored, no false grief.)
    local GD_PLACED = {
        {kw="boogie",name="Boogie Bomb"},
        {kw="body swap",name="Body Swap Potion"},{kw="bodyswap",name="Body Swap Potion"},
        {kw="trap",name="Trap"},
        {kw="subspace",name="Subspace Mine"},
        {kw="beehive",name="Beehive"},
        {kw="sentry",name="All-Seeing Sentry"},
        {kw="rage table",name="Rage Table"},
        {kw="gummy",name="Gummy Slap"},
        {kw="medusa",name="Medusa's Head"},
    }
    Workspace.DescendantAdded:Connect(function(obj)
        local nm=obj.Name:lower()
        local hit=nil
        for _,g in ipairs(GD_PLACED) do
            if nm:find(g.kw,1,true) then hit=g.name; break end
        end
        if not hit then return end
        task.defer(function()
            pcall(function()
                local myHrp=player.Character and player.Character:FindFirstChild("HumanoidRootPart")
                if not myHrp then return end
                local pos
                if obj:IsA("BasePart") then pos=obj.Position
                elseif obj:IsA("Model") then pos=obj:GetPivot().Position
                else return end
                if (pos-myHrp.Position).Magnitude>60 then return end
                -- equipped gear (inside a character) is not a placement
                for _,pl in ipairs(Players:GetPlayers()) do
                    if pl.Character and obj:IsDescendantOf(pl.Character) then return end
                end
                gdReport(hit)
            end)
        end)
    end)
end)

-- LAZY UI LOADING
if _G.addLazyUI then
    _G.addLazyUI(main, not Config.AutoCloseOnExec)
    _G.addLazyUI(actionSettingsPanel, false)
    _G.addLazyUI(tpSpeedSettingsPanel, false)
end

immediatePanels = {
    ["Steal Target"] = true,
    ["Steal Panel"] = true,
    ["Invisible Steal Panel"] = true
}

for name, panel in pairs(panels) do
    if not string.match(name, "Body$") then
        -- _G._MynxxPanelVis survives re-executes; override Config if it has a value
        local fromG = _G._MynxxPanelVis[name]
        local targetVis
        if fromG ~= nil then
            targetVis = fromG
            Config.Visibilities[name] = fromG  -- sync back so Config is consistent
        else
            if Config.Visibilities[name] ~= nil then targetVis = Config.Visibilities[name] else targetVis = true end
        end
        if immediatePanels[name] then
            panel.Visible = targetVis
        elseif _G.addLazyUI then
            _G.addLazyUI(panel, targetVis, false, name)
        end
    end
end

-- ACTIONS PANEL 
function rebuildActions()
    clearBody(panels["ActionsBody"])
    if actionConfig["Ragdoll Self (R)"] then makeQuickButton(panels["ActionsBody"],"Ragdoll Self (R)",function() requestRagdollSelf() end) end
    makeQuickButton(panels["ActionsBody"], "Teleport", function()
    task.spawn(function() pcall(doGrabbleVelocityTP) end)
end)
    if actionConfig["Rejoin PS"] then makeQuickButton(panels["ActionsBody"],"Rejoin PS",function() TeleportService:Teleport(game.PlaceId,player) end) end
    makeQuickButton(panels["ActionsBody"], "Drop Brainrot", function() runDropBrainrot() end)
    if actionConfig["Rejoin"] then makeQuickButton(panels["ActionsBody"],"Rejoin",function()
        -- Rejoin the SAME server instance via JobId. Retries only (no public
        -- fallback), so it can never land you on a different server.
        local jobId = game.JobId
        task.spawn(function()
            if not jobId or jobId == "" then return end
            for _ = 1, 4 do
                local ok = pcall(function() TeleportService:TeleportToPlaceInstance(game.PlaceId, jobId, player) end)
                if ok then break end
                task.wait(1.5)
            end
        end)
    end) end
    if actionConfig["Kick (Y)"] then makeQuickButton(panels["ActionsBody"],"Kick (Y)",function() kickPlayer() end) end
    if actionConfig["Self Kick"] then makeQuickButton(panels["ActionsBody"],"Self Kick",function() pcall(function() player:Kick("You have been kicked.") end) end) end
    if actionConfig["Kick To Private"] then makeQuickButton(panels["ActionsBody"],"Kick To Private",function() 
        if PrivateServerCode and PrivateServerCode ~= "" then
            task.delay(0.2, function()
                pcall(function() game:GetService("ExperienceService"):LaunchExperience({placeId=game.PlaceId,linkCode=PrivateServerCode}) end)
            end)
        end
    end) end
    if actionConfig["Reset (X)"] then makeQuickButton(panels["ActionsBody"],"Reset (X)",function()
        if _G.MynxxRegenerate then
            pcall(_G.MynxxRegenerate)
        else
            pcall(executeReset)
        end
    end,Theme.SoftAccentHover) end
    if actionConfig["Anti Ragdoll"] then makeSyncStateRow(panels["ActionsBody"],"Anti Ragdoll:","Anti Ragdoll",function(on) if on then startAntiRagdoll() else stopAntiRagdoll() end end) end
    if actionConfig["Infinite Jump"] then makeSyncStateRow(panels["ActionsBody"],"Infinite Jump:","Infinite Jump",function(on) setInfiniteJump(on) end) end
    if actionConfig["Carpet Speed"] then makeSyncStateRow(panels["ActionsBody"],"Carpet Speed:","Carpet Speed",function(on) setCarpetSpeed(on) end) end
    makeQuickButton(panels["ActionsBody"],"Settings",function()
        if actionSettingsPanel.Visible then closeAnim(actionSettingsPanel) else openAnim(actionSettingsPanel) end
    end,Theme.SoftAccent)
end

function rebuildActionSettings()
    clearBody(actionSettingsBody)
    for actionName,enabled in pairs(actionConfig) do
        local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,34); row.BackgroundTransparency=1; row.Parent=actionSettingsBody
        local label=Instance.new("TextLabel"); label.Size=UDim2.new(1,-84,1,0); label.Position=UDim2.new(0,4,0,0); label.BackgroundTransparency=1; label.Text=actionName; label.TextColor3=Theme.Text; label.Font=Enum.Font.GothamSemibold; label.TextSize=11; label.TextXAlignment=Enum.TextXAlignment.Left; label.TextTruncate=Enum.TextTruncate.AtEnd; label.Parent=row
        local btn=Instance.new("TextButton"); btn.Name="WhiteTextBtn"; btn.Size=UDim2.new(0,72,0,30); btn.Position=UDim2.new(1,-74,0.5,-15); btn.BackgroundColor3=enabled and Theme.Green or Theme.ToggleOff2; btn.BackgroundTransparency=UITransparency.Toggle; btn.Text=enabled and "ON" or "OFF"; btn.TextColor3=Color3.new(1,1,1); btn.Font=Enum.Font.GothamBlack; btn.TextSize=12; btn.AutoButtonColor=false; btn.Parent=row; corner(btn,6)
        btn.MouseButton1Click:Connect(function()
            actionConfig[actionName]=not actionConfig[actionName]; Config.actions[actionName]=actionConfig[actionName]; saveConfig()
            btn.BackgroundColor3=actionConfig[actionName] and Theme.Green or Theme.ToggleOff2; btn.Text=actionConfig[actionName] and "ON" or "OFF"
            rebuildActions()
        end)
    end
    makeQuickButton(actionSettingsBody,"Close",function() closeAnim(actionSettingsPanel) end,Theme.SoftAccentHover)
end

function rebuildTpSpeedSettings()
    clearBody(tpSpeedSettingsBody)
    makeMainSliderWithInput(tpSpeedSettingsBody, "Grabble TP Speed", 50, 600, Config.TpSettings.GrabbleTPSpeed or 450, function(v) Config.TpSettings.GrabbleTPSpeed=v; saveConfig(); if _G.MynxxSetCarpetSpeed then pcall(_G.MynxxSetCarpetSpeed, v) end end)
    makeMainSliderWithInput(tpSpeedSettingsBody, "Walk To Brainrot Speed", 50, 300, Config.TpSettings.WalkTPSpeed or 190, function(v) Config.TpSettings.WalkTPSpeed=v; saveConfig() end)
    makeMainSliderWithInput(tpSpeedSettingsBody, "Clone Delay", 0.05, 2.0, Config.TpSettings.CloneDelayVal or 0.1, function(v) Config.TpSettings.CloneDelayVal=v; saveConfig() end, "s")
    makeQuickButton(tpSpeedSettingsBody, "Close", function() closeAnim(tpSpeedSettingsPanel) end, Theme.SoftAccentHover)
end

do
    -- INVISIBLE STEAL PANEL
    _G.InvisStealAngle = Config.InvisStealAngle or 180
    _G.SinkSliderValue = Config.SinkSliderValue or 8
    setWalkSpeedValue(Config.WalkSpeedValue or 28)

    regToggle("Auto Recover Lagback", Config.AutoRecoverLagback)
    regToggle("Auto Invis During Steal", Config.AutoInvisDuringSteal)
    regToggle("WalkSpeed", Config.WalkSpeedEnabled)

    makeSyncStateRow(panels["InvisStealBody"], "Enabled:", "Invisible Steal", function(on)
        if _G.toggleInvisibleSteal then pcall(_G.toggleInvisibleSteal) end
    end)
    makeQuickSlider(panels["InvisStealBody"], "Rotation", 0, 360, Config.InvisStealAngle or 180, function(v)
        _G.InvisStealAngle = v; Config.InvisStealAngle = v; saveConfig()
    end)
    makeQuickSlider(panels["InvisStealBody"], "Depth", 0, 18, Config.SinkSliderValue or 8, function(v)
        _G.SinkSliderValue = v; Config.SinkSliderValue = v; saveConfig()
    end)
    makeQuickSlider(panels["InvisStealBody"], "Walk Speed", 15, 29, Config.WalkSpeedValue or 28, function(v)
        setWalkSpeedValue(v)
    end)
    makeSyncStateRow(panels["InvisStealBody"], "Auto Recover:", "Auto Recover Lagback", function(on)
        _G.AutoRecoverLagback = on; Config.AutoRecoverLagback = on; saveConfig()
    end)
    makeSyncStateRow(panels["InvisStealBody"], "Auto Invis:", "Auto Invis During Steal", function(on)
        _G.AutoInvisDuringSteal = on; Config.AutoInvisDuringSteal = on; saveConfig()
    end)
    makeSyncStateRow(panels["InvisStealBody"], "WalkSpeed:", "WalkSpeed", function(on)
        setWalkSpeedEnabled(on)
    end)

    _G.updateMovementPanelLabels = function() end
    if Config.WalkSpeedEnabled then
        task.defer(function() setWalkSpeedEnabled(true) end)
    end


-- ADMIN QUICK PANEL
function spamPlayerBaseOwner(targetPlayer)
    if not targetPlayer then return 0 end
    local cmdList = Config.SpamBaseOwnerOrder or AP_ALL_COMMANDS
    
    if Config.SpamBaseOwnerSingleCommand then
        local startIndex = _G.SpamBaseOwnerIndex or 1
        if startIndex > #cmdList then startIndex = 1 end
        
        local picked = nil
        local attempts = 0
        while attempts < #cmdList do
            local idx = ((startIndex - 1 + attempts) % #cmdList) + 1
            local cmd = cmdList[idx]
            if Config.SpamBaseOwnerCommands and Config.SpamBaseOwnerCommands[cmd] and not apIsOnCooldown(cmd) then
                picked = cmd
                _G.SpamBaseOwnerIndex = idx + 1
                break
            end
            attempts = attempts + 1
        end
        
        if picked then
            runAdminCommand(targetPlayer, picked)
            return 1
        end
        return 0
    else
        local activeCmds = {}
        for _, cmd in ipairs(cmdList) do
            if Config.SpamBaseOwnerCommands and Config.SpamBaseOwnerCommands[cmd] and not apIsOnCooldown(cmd) then
                table.insert(activeCmds, cmd)
            end
        end
        -- 1 command per second until every active command has been used
        task.spawn(function()
            for i, cmd in ipairs(activeCmds) do
                if not targetPlayer or not targetPlayer.Parent then break end
                runAdminCommand(targetPlayer, cmd)
                if i < #activeCmds then task.wait(0.01) end
            end
        end)
        return #activeCmds
    end
end

-- ADMIN QUICK PANEL
function spamBaseOwner()
    local char = LocalPlayer.Character
    local hrp = char and char:FindFirstChild("HumanoidRootPart")
    if not hrp then
        ShowNotification("SPAM OWNER", "No character found")
        return
    end
    local nearestPlot = nil
    local nearestDist = math.huge
    local Plots = Workspace:FindFirstChild("Plots")
    if Plots then
        for _, plot in ipairs(Plots:GetChildren()) do
            local sign = plot:FindFirstChild("PlotSign")
            if sign then
                local yourBase = sign:FindFirstChild("YourBase")
                if not yourBase or not yourBase.Enabled then
                    local signPos = (sign:IsA("BasePart") and sign.Position)
                        or (sign.PrimaryPart and sign.PrimaryPart.Position)
                    if not signPos then
                        local part = sign:FindFirstChildWhichIsA("BasePart", true)
                        signPos = part and part.Position
                    end
                    if signPos then
                        local dist = (hrp.Position - signPos).Magnitude
                        if dist < nearestDist then
                            nearestDist = dist
                            nearestPlot = plot
                        end
                    end
                end
            end
        end
    end
    if not nearestPlot then
        ShowNotification("SPAM OWNER", "No nearby base found")
        return
    end
    local targetPlayer = nil
    local ok, Synchronizer = pcall(require, ReplicatedStorage:WaitForChild("Packages"):WaitForChild("Synchronizer"))
    if ok and Synchronizer then
        local ch = _G.XenSyncGet(nearestPlot.Name)
        if ch then
            local owner = ch:Get("Owner")
            if owner then
                if (typeof(owner) == "Instance") and owner:IsA("Player") then
                    targetPlayer = owner
                elseif (type(owner) == "table") and owner.Name then
                    targetPlayer = Players:FindFirstChild(owner.Name)
                end
            end
        end
    end
    if not targetPlayer then
        local sign = nearestPlot:FindFirstChild("PlotSign")
        local textLabel = sign
            and sign:FindFirstChild("SurfaceGui")
            and sign.SurfaceGui:FindFirstChild("Frame")
            and sign.SurfaceGui.Frame:FindFirstChild("TextLabel")
        if textLabel then
            local baseText = textLabel.Text
            local nickname = (baseText and baseText:match("^(.-)'")) or baseText
            if nickname then
                for _, p in ipairs(Players:GetPlayers()) do
                    if (p.DisplayName == nickname) or (p.Name == nickname) then
                        targetPlayer = p
                        break
                    end
                end
            end
        end
    end
    if not targetPlayer or (targetPlayer == LocalPlayer) then
        ShowNotification("SPAM OWNER", "Owner not found or is you")
        return
    end
    if isPlayerBlacklisted(targetPlayer) then
        ShowNotification("SPAM OWNER", targetPlayer.DisplayName .. " is blacklisted")
        return
    end
    ShowNotification("SPAM OWNER", "Spamming " .. targetPlayer.DisplayName)
    local sentCount = spamPlayerBaseOwner(targetPlayer)
    ShowNotification("SPAM OWNER", "Sent " .. tostring(sentCount) .. " commands")
end

makeSyncStateRow(panels["AdminBody"],"Click to AP:","Click to AP",function(on) Config.ClickToAP=on; saveConfig() end)
makeSyncStateRow(panels["AdminBody"],"Proximity:","Proximity",function(on) setProximityAP(on) end)
makeQuickSlider(panels["AdminBody"],"Distance",1,50,Config.ProximityRange or 15,function(v) Config.ProximityRange=v; saveConfig() end," studs")
makeQuickButton(panels["AdminBody"],"Spam Base Owner",spamBaseOwner)

-- COOLDOWN PANEL
LazyInit("Cooldown Panel", function() -- COOLDOWN PANEL SCOPE
cooldownLabels={}
for _,item in ipairs({"jail","rocket","inverse","ragdoll","jumpscare","tiny","balloon","morph","nightvision"}) do
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,24); row.BackgroundTransparency=1; row.Parent=panels["CooldownBody"]
    local left=Instance.new("TextLabel"); left.Size=UDim2.new(0.58,0,1,0); left.Position=UDim2.new(0,6,0,0); left.BackgroundTransparency=1; left.Text=item:sub(1,1):upper()..item:sub(2); left.TextColor3=Theme.Text; left.Font=Enum.Font.GothamBold; left.TextSize=12; left.TextXAlignment=Enum.TextXAlignment.Left; left.Parent=row
    local right=Instance.new("TextLabel"); right.Size=UDim2.new(0.36,0,1,0); right.Position=UDim2.new(0.62,0,0,0); right.BackgroundTransparency=1; right.Text="READY"; right.TextColor3=Theme.Green; right.Font=Enum.Font.GothamBlack; right.TextSize=11; right.TextXAlignment=Enum.TextXAlignment.Right; right.Parent=row
    cooldownLabels[item]=right
end
task.spawn(function() while true do task.wait(0.5); for cmd,label in pairs(cooldownLabels) do local rem=apGetRemaining(cmd); if rem>0 then label.Text=string.format("%.0fs",rem); label.TextColor3=Theme.Red else label.Text="READY"; label.TextColor3=Theme.Green end end end end)
end) -- END COOLDOWN PANEL SCOPE (LazyInit)

-- STEAL PANEL
makeSyncStateRow(panels["StealBody"],"Auto Steal:","Auto Steal",function(on)
    autoStealEnabled=on; Config.AutoStealEnabled=on; saveConfig()
    -- Mynxx Clone-TP engine owns the auto-steal loop unconditionally.
    if _G.MynxxAutoSteal then pcall(_G.MynxxAutoSteal, on) end
end)
makeSyncStateRow(panels["StealBody"],"Steal Highest:","Steal Highest",function(on) if on then setStealMode("Highest") end end)
makeSyncStateRow(panels["StealBody"],"Steal Priority:","Steal Priority",function(on) if on then setStealMode("Priority") end end)
makeSyncStateRow(panels["StealBody"],"Steal Nearest:","Steal Nearest",function(on) if on then setStealMode("Nearest") end end)
makeSyncStateRow(panels["StealBody"],"Auto Buy:","Auto Buy",function(on)
    if toggleAutoBuy then toggleAutoBuy(on)
    else warn("[Mynxx] Auto Buy not ready yet -- try again in a sec") end
end)
makeSyncStateRow(panels["StealBody"],"Auto Kick:","Auto Kick",function(on) Config.AutoKickOnSteal=on; saveConfig() end)
makeSyncStateRow(panels["StealBody"],"Auto Destroy Turrets:","Auto Destroy Turrets",function(on) Config.AutoDestroyTurrets=on; saveConfig() end)


-- STEAL TARGET (0.4s refresh) -- NOT lazy: panel must populate on load
do -- STEAL TARGET SCOPE

-- MINI ICONES 3D: com monitoramento contínuo
local _miniVPCache = {} -- [uid] = {frame, clone, center, radius, frontYaw, oscT, camera, pet, checking}

-- Função para tentar clonar o modelo
local function _tryCloneModel(entry, pet)
    if entry.clone and entry.clone.Parent then return true end
    local sourceModel = findBestAnimalModel(pet)
    if not sourceModel then return false end

    pcall(function()
        local clone = sourceModel:Clone()
        for _,desc in ipairs(clone:GetDescendants()) do
            if desc:IsA("Script") or desc:IsA("LocalScript") or desc:IsA("ModuleScript") or desc:IsA("Sound") or desc:IsA("SoundGroup") or desc:IsA("BodyMover") or desc:IsA("Constraint") then
                desc:Destroy()
            elseif desc:IsA("BasePart") then
                desc.Anchored = true
                desc.CanCollide = false
            elseif desc:IsA("Animator") then
                for _,tr in ipairs(desc:GetPlayingAnimationTracks()) do tr:Stop(0) end
            end
        end
        local root = clone:FindFirstChild("HumanoidRootPart") or clone.PrimaryPart or clone:FindFirstChildWhichIsA("BasePart")
        if root then
            local look = root.CFrame.LookVector
            entry.frontYaw = math.deg(math.atan2(look.X, look.Z))
        end
        local boundCF = clone:GetBoundingBox()
        clone:PivotTo(clone:GetPivot() + (Vector3.new(0,0,0) - boundCF.Position))
        clone.Parent = entry.frame
        entry.clone = clone
        local newCF, sz = clone:GetBoundingBox()
        entry.center = newCF.Position
        entry.radius = math.max(math.max(sz.X,sz.Y,sz.Z)*0.55+0.5, 2.0)
    end)
    return entry.clone and entry.clone.Parent or false
end

-- Cria um entry vazio (sem clone) e inicia o monitoramento
local function _createEntry(pet)
    local frame = Instance.new("ViewportFrame")
    frame.BackgroundColor3 = Color3.fromRGB(10,10,12)
    frame.BackgroundTransparency = 0.1
    frame.BorderSizePixel = 0
    frame.LightDirection = Vector3.new(-1,-1,-1)
    frame.Ambient = Color3.fromRGB(180,180,200)
    corner(frame, 6)
    local cam = Instance.new("Camera")
    cam.FieldOfView = 50
    cam.Parent = frame
    frame.CurrentCamera = cam

    local entry = {
        frame = frame,
        clone = nil,
        center = Vector3.new(0,0,0),
        radius = 2.5,
        frontYaw = 0,
        oscT = math.random() * 10,
        camera = cam,
        pet = pet,
        checking = false
    }

    -- Tenta clonar imediatamente
    _tryCloneModel(entry, pet)

    -- Se falhou, inicia um verificador periódico
    if not entry.clone then
        entry.checking = true
        task.spawn(function()
            local uid = tostring(pet.uid)
            while entry.checking do
                local currentEntry = _miniVPCache[uid]
                if not currentEntry or currentEntry ~= entry then break end
                if currentEntry.clone and currentEntry.clone.Parent then
                    currentEntry.checking = false
                    break
                end
                if _tryCloneModel(currentEntry, pet) then
                    currentEntry.checking = false
                    break
                end
                task.wait(0.5) -- tenta a cada 0.5s até conseguir
            end
        end)
    end

    return entry
end

-- Pega (ou cria) o mini-viewport de um pet
local function getMiniIconFor(pet)
    if Config and Config.Steal3DViewer == false then
        local blank = Instance.new("Frame")
        blank.BackgroundTransparency = 1
        blank.Size = UDim2.new(0, 34, 0, 34)
        return blank
    end
    local uid = tostring(pet.uid)
    local entry = _miniVPCache[uid]

    if entry then
        if entry.clone and entry.clone.Parent then
            return entry.frame
        end
        return entry.frame
    end

    entry = _createEntry(pet)
    _miniVPCache[uid] = entry
    return entry.frame
end

-- Heartbeat para animar os clones existentes
RunService.Heartbeat:Connect(function(dt)
    for uid, entry in pairs(_miniVPCache) do
        if entry.clone and entry.clone.Parent and entry.frame.Parent then
            entry.oscT = entry.oscT + dt
            local r = math.rad(entry.frontYaw + math.sin(entry.oscT * 1.3) * 14)
            local dist = entry.radius * 2.6
            pcall(function()
                entry.camera.CFrame = CFrame.new(
                    entry.center + Vector3.new(math.sin(r) * dist, entry.radius * 0.35, math.cos(r) * dist),
                    entry.center
                )
            end)
        end
    end
end)

-- Remove do cache os mini-viewports de pets que sumiram da lista
local function _miniPruneCache(seenUIDs)
    for uid, entry in pairs(_miniVPCache) do
        if not seenUIDs[uid] then
            if entry.checking then entry.checking = false end
            pcall(function() entry.frame:Destroy() end)
            _miniVPCache[uid] = nil
        end
    end
end

-- STEAL TARGET (0.4s refresh) -- NOT lazy: panel must populate on load
do -- STEAL TARGET SCOPE

-- MINI ICONES 3D: com monitoramento contínuo
local _miniVPCache = {} -- [uid] = {frame, clone, center, radius, frontYaw, oscT, camera, pet, checking}

-- Função para tentar clonar o modelo
local function _tryCloneModel(entry, pet)
    if entry.clone and entry.clone.Parent then return true end
    local sourceModel = findBestAnimalModel(pet)
    if not sourceModel then return false end

    pcall(function()
        local clone = sourceModel:Clone()
        for _,desc in ipairs(clone:GetDescendants()) do
            if desc:IsA("Script") or desc:IsA("LocalScript") or desc:IsA("ModuleScript") or desc:IsA("Sound") or desc:IsA("SoundGroup") or desc:IsA("BodyMover") or desc:IsA("Constraint") then
                desc:Destroy()
            elseif desc:IsA("BasePart") then
                desc.Anchored = true
                desc.CanCollide = false
            elseif desc:IsA("Animator") then
                for _,tr in ipairs(desc:GetPlayingAnimationTracks()) do tr:Stop(0) end
            end
        end
        local root = clone:FindFirstChild("HumanoidRootPart") or clone.PrimaryPart or clone:FindFirstChildWhichIsA("BasePart")
        if root then
            local look = root.CFrame.LookVector
            entry.frontYaw = math.deg(math.atan2(look.X, look.Z))
        end
        local boundCF = clone:GetBoundingBox()
        clone:PivotTo(clone:GetPivot() + (Vector3.new(0,0,0) - boundCF.Position))
        clone.Parent = entry.frame
        entry.clone = clone
        local newCF, sz = clone:GetBoundingBox()
        entry.center = newCF.Position
        entry.radius = math.max(math.max(sz.X,sz.Y,sz.Z)*0.55+0.5, 2.0)
    end)
    return entry.clone and entry.clone.Parent or false
end

-- Cria um entry vazio (sem clone) e inicia o monitoramento
local function _createEntry(pet)
    local frame = Instance.new("ViewportFrame")
    frame.BackgroundColor3 = Color3.fromRGB(10,10,12)
    frame.BackgroundTransparency = 0.1
    frame.BorderSizePixel = 0
    frame.LightDirection = Vector3.new(-1,-1,-1)
    frame.Ambient = Color3.fromRGB(180,180,200)
    corner(frame, 6)
    local cam = Instance.new("Camera")
    cam.FieldOfView = 50
    cam.Parent = frame
    frame.CurrentCamera = cam

    local entry = {
        frame = frame,
        clone = nil,
        center = Vector3.new(0,0,0),
        radius = 2.5,
        frontYaw = 0,
        oscT = math.random() * 10,
        camera = cam,
        pet = pet,
        checking = false,
        watcher = nil,
        lastSig = (pet.petName or "") .. "|" .. (pet.mutation or "")
    }

    -- Tenta clonar imediatamente
    _tryCloneModel(entry, pet)

    -- Se falhou, inicia um verificador periódico
    if not entry.clone then
        entry.checking = true
        local uid = tostring(pet.uid)
        entry.watcher = task.spawn(function()
            while entry.checking do
                local currentEntry = _miniVPCache[uid]
                if not currentEntry or currentEntry ~= entry then break end
                if currentEntry.clone and currentEntry.clone.Parent then
                    currentEntry.checking = false
                    break
                end
                if _tryCloneModel(currentEntry, entry.pet) then
                    currentEntry.checking = false
                    break
                end
                task.wait(0.3)
            end
        end)
    end

    return entry
end

-- Pega (ou cria) o mini-viewport de um pet
local function getMiniIconFor(pet)
    if Config and Config.Steal3DViewer == false then
        local blank = Instance.new("Frame")
        blank.BackgroundTransparency = 1
        blank.Size = UDim2.new(0, 34, 0, 34)
        return blank
    end
    local uid = tostring(pet.uid)
    local entry = _miniVPCache[uid]
    local sig = (pet.petName or "") .. "|" .. (pet.mutation or "")

    if entry then
        if sig ~= entry.lastSig then
            if entry.checking then entry.checking = false end
            if entry.watcher then task.cancel(entry.watcher); entry.watcher = nil end
            if entry.clone then pcall(function() entry.clone:Destroy() end); entry.clone = nil end
            entry.lastSig = sig
            entry.pet = pet
            _tryCloneModel(entry, pet)
            if not entry.clone then
                entry.checking = true
                entry.watcher = task.spawn(function()
                    while entry.checking do
                        local currentEntry = _miniVPCache[uid]
                        if not currentEntry or currentEntry ~= entry then break end
                        if currentEntry.clone and currentEntry.clone.Parent then
                            currentEntry.checking = false
                            break
                        end
                        if _tryCloneModel(currentEntry, entry.pet) then
                            currentEntry.checking = false
                            break
                        end
                        task.wait(0.3)
                    end
                end)
            end
        elseif not (entry.clone and entry.clone.Parent) then
            if _tryCloneModel(entry, pet) then
                entry.checking = false
            elseif not entry.checking then
                entry.checking = true
                entry.watcher = task.spawn(function()
                    while entry.checking do
                        local currentEntry = _miniVPCache[uid]
                        if not currentEntry or currentEntry ~= entry then break end
                        if currentEntry.clone and currentEntry.clone.Parent then
                            currentEntry.checking = false
                            break
                        end
                        if _tryCloneModel(currentEntry, entry.pet) then
                            currentEntry.checking = false
                            break
                        end
                        task.wait(0.3)
                    end
                end)
            end
        end
        return entry.frame
    end

    entry = _createEntry(pet)
    _miniVPCache[uid] = entry
    return entry.frame
end

-- Heartbeat para animar os clones existentes
RunService.Heartbeat:Connect(function(dt)
    for uid, entry in pairs(_miniVPCache) do
        if entry.clone and entry.clone.Parent and entry.frame.Parent then
            entry.oscT = entry.oscT + dt
            local r = math.rad(entry.frontYaw + math.sin(entry.oscT * 1.3) * 14)
            local dist = entry.radius * 2.6
            pcall(function()
                entry.camera.CFrame = CFrame.new(
                    entry.center + Vector3.new(math.sin(r) * dist, entry.radius * 0.35, math.cos(r) * dist),
                    entry.center
                )
            end)
        end
    end
end)

-- Remove do cache os mini-viewports de pets que sumiram da lista
local function _miniPruneCache(seenUIDs)
    for uid, entry in pairs(_miniVPCache) do
        if not seenUIDs[uid] then
            if entry.checking then entry.checking = false end
            pcall(function() entry.frame:Destroy() end)
            _miniVPCache[uid] = nil
        end
    end
end

-- ============================================================
-- FUNÇÃO PRINCIPAL QUE ATUALIZA O PAINEL (COM makeTargetRow DENTRO)
-- ============================================================
function refreshTargetPanel()
    -- Desanexa os mini-viewports
    for _, entry in pairs(_miniVPCache) do entry.frame.Parent = nil end
    clearBody(panels["TargetBody"])
    
    local cache = get_all_pets()
    if not cache or #cache == 0 then
        local l = Instance.new("TextLabel")
        l.Size = UDim2.new(1,-4,0,24)
        l.BackgroundTransparency = 1
        l.Text = "Scanning..."
        l.TextColor3 = Theme.Dim
        l.Font = Enum.Font.GothamSemibold
        l.TextSize = 11
        l.TextXAlignment = Enum.TextXAlignment.Center
        l.Parent = panels["TargetBody"]
        return
    end

    -- ════════════════════════════════════════════════
    -- ORDENA PELO MAIOR mpsValue (1.1B vem primeiro)
    -- ════════════════════════════════════════════════
    table.sort(cache, function(a, b)
        return (a.mpsValue or 0) > (b.mpsValue or 0)
    end)

    -- Mapa de prioridade (apenas para destaque visual)
    local prioSet = {}
    for _, pName in ipairs(priorityList) do prioSet[pName:lower()] = true end

    -- ============================================================
    -- FUNÇÃO QUE CRIA CADA LINHA (DEFINIDA AQUI DENTRO)
    -- ============================================================
    local function makeTargetRow(pet, displayIndex, isPriority)
        local isSelected = (selectedTargetUID == pet.uid)
        local isManual = (manuallySelectedUID == pet.uid)
        local row = Instance.new("Frame")
        row.Size = UDim2.new(1,-4,0,44)
        row.BackgroundColor3 = isSelected and Theme.RowHover or Theme.Panel
        row.BackgroundTransparency = isPriority and UITransparency.TargetRow or UITransparency.TargetRowAlt
        row.Parent = panels["TargetBody"]
        corner(row, 6)
        if isSelected or isManual then
            local str = Instance.new("UIStroke", row)
            str.Color = isManual and Color3.fromRGB(56, 214, 110) or Theme.Accent
            str.Thickness = isManual and 1.75 or 1.25
        elseif isPriority then
            local str = Instance.new("UIStroke", row)
            str.Color = Color3.fromRGB(255, 90, 120)
            str.Thickness = 0.75
            str.Transparency = 0.3
        end
        
        local rankBox = Instance.new("Frame")
        rankBox.Size = UDim2.new(0, 26, 0, 26)
        rankBox.Position = UDim2.new(0, 8, 0.5, -13)
        rankBox.BackgroundColor3 = Color3.fromRGB(70, 70, 74)
        rankBox.BorderSizePixel = 0
        rankBox.Parent = row
        corner(rankBox, 4)
        
        local n = Instance.new("TextLabel")
        n.Size = UDim2.new(1,0,1,0)
        n.BackgroundTransparency = 1
        n.Text = "#"..displayIndex
        n.TextColor3 = Color3.new(1,1,1)
        n.Font = Enum.Font.GothamBold
        n.TextSize = 12
        n.Parent = rankBox

        -- Mini icone 3D
        local miniIcon = getMiniIconFor(pet)
        miniIcon.Size = UDim2.new(0, 34, 0, 34)
        miniIcon.Position = UDim2.new(0, 40, 0.5, -17)
        miniIcon.Parent = row

        local nm = Instance.new("TextLabel")
        nm.Size = UDim2.new(1,-86,0,18)
        nm.Position = UDim2.new(0,80,0,5)
        nm.BackgroundTransparency = 1
        nm.RichText = true
        local _mutTag = ""
        local _mut = pet.mutation
        if _mut and _mut ~= "None" and _mut ~= "none" then
            local _mutColors = {
                Gold="rgb(255,222,89)", Diamond="rgb(37,196,254)", Bloodrot="rgb(145,0,27)",
                Rainbow="rgb(255,0,251)", Candy="rgb(192,192,192)", Lava="rgb(255,149,0)",
                Galaxy="rgb(170,60,255)", YinYang="rgb(220,220,220)", Radioactive="rgb(104,245,0)",
                Cursed="rgb(245,56,56)", Divine="rgb(255,209,59)", Cyber="rgb(121,219,255)",
                Phantom="rgb(180,180,180)", NORMAL="rgb(160,160,160)",
            }
            local _mc = _mutColors[_mut] or "rgb(200,100,255)"
            _mutTag = "<font color='" .. _mc .. "'>[" .. _mut .. "]</font> "
        end
        nm.Text = _mutTag .. (pet.petName or "?")
        nm.TextColor3 = isPriority and Color3.new(1,1,1) or Theme.Text
        nm.Font = isPriority and Enum.Font.GothamBold or Enum.Font.GothamSemibold
        nm.TextSize = 13
        nm.TextXAlignment = Enum.TextXAlignment.Left
        nm.TextTruncate = Enum.TextTruncate.AtEnd
        nm.Parent = row
        
        local gn = Instance.new("TextLabel")
        gn.Size = UDim2.new(1,-86,0,14)
        gn.Position = UDim2.new(0,80,0,23)
        gn.BackgroundTransparency = 1
        gn.RichText = true
        local ownerText = pet.owner and (' <font color="#999999">| @' .. tostring(pet.owner) .. '</font>') or ""
        gn.Text = '<font color="#38D66E">Gen: ' .. (pet.mpsText or "") .. '</font>' .. ownerText
        gn.TextColor3 = Color3.new(1,1,1)
        gn.Font = Enum.Font.GothamMedium
        gn.TextSize = 11
        gn.TextXAlignment = Enum.TextXAlignment.Left
        gn.TextTruncate = Enum.TextTruncate.AtEnd
        gn.Parent = row

        local overlay = Instance.new("TextButton")
        overlay.Size = UDim2.new(1,0,1,0)
        overlay.BackgroundTransparency = 1
        overlay.Text = ""
        overlay.ZIndex = 8
        overlay.Parent = row
        overlay.MouseButton1Click:Connect(function()
            if manuallySelectedUID == pet.uid then
                manuallySelectedUID = nil
                selectedTargetUID = nil
                SharedState.SelectedPetData = nil
                if _G._MynxxUpdateStViewer then pcall(_G._MynxxUpdateStViewer, nil) end
            else
                selectedTargetUID = pet.uid
                manuallySelectedUID = pet.uid
                SharedState.SelectedPetData = pet
                if pet then SharedState.LastTargetedPetMpsValue = pet.mpsValue or 0 end
                if _G._MynxxUpdateStViewer then pcall(_G._MynxxUpdateStViewer, pet) end

                task.spawn(function()
                    local plot, _ = getPlayerBaseInfo(LocalPlayer)
                    if plot then return end
                    if _G.Mynxx_ExecuteManualTP then pcall(_G.Mynxx_ExecuteManualTP) end
                end)
            end
            refreshTargetPanel()
        end)
    end

    -- Cria as linhas na ordem do cache (já ordenado)
    local idx = 0
    local seenUIDs = {}
    for _, pet in ipairs(cache) do
        idx = idx + 1
        local isPriority = (pet.petName and prioSet[pet.petName:lower()]) or false
        makeTargetRow(pet, idx, isPriority)
        seenUIDs[tostring(pet.uid)] = true
    end

    _miniPruneCache(seenUIDs)
end

-- Inicia o loop de atualização do painel a cada 0.4 segundos
task.spawn(function() while true do task.wait(0.4); refreshTargetPanel() end end)

end -- END STEAL TARGET SCOPE

-- Inicia o loop de atualização do painel a cada 0.4 segundos
task.spawn(function() while true do task.wait(0.4); refreshTargetPanel() end end)

end -- END STEAL TARGET SCOPE

-- ============================================================
-- ADMIN PANEL UI
-- ============================================================
LazyInit("Admin Panel UI", function() -- ADMIN PANEL UI SCOPE
    pcall(function() local e=playerGui:FindFirstChild("XiAdminPanel"); if e then e:Destroy() end end)
    apGui=Instance.new("ScreenGui"); apGui.Name="XiAdminPanel"; apGui.ResetOnSpawn=false; apGui.IgnoreGuiInset=true; apGui.DisplayOrder=9999998; apGui.ZIndexBehavior=Enum.ZIndexBehavior.Sibling; apGui.Parent=playerGui
    apGui.Enabled = (Config.AdminPanelUI == true)
    apOuter=Instance.new("Frame"); apOuter.Name="Frame"; apOuter.BackgroundTransparency=1; apOuter.BorderSizePixel=0; apOuter.Size=UDim2.fromOffset(480,0); apOuter.AutomaticSize=Enum.AutomaticSize.Y; apOuter.Position=UDim2.new(0.18,0,0.57,0); apOuter.ZIndex=10; apOuter.ClipsDescendants=true; apOuter.Parent=registerScreenGui(apGui)
    apBG=Instance.new("Frame"); apBG.BackgroundTransparency=1; apBG.BorderSizePixel=0; apBG.Position=UDim2.fromOffset(-3,-2); apBG.Size=UDim2.new(1,6,1,4); apBG.ZIndex=0; apBG.Parent=apOuter; corner(apBG,8)
    apTop=Instance.new("Frame"); apTop.BackgroundTransparency=1; apTop.BorderSizePixel=0; apTop.Size=UDim2.new(1,0,0,16); apTop.Parent=apOuter; corner(apTop,3)

    makeDraggable(apOuter,apTop,"AdminPanel"); applySavedPosition("AdminPanel",apOuter);

    apList=Instance.new("Frame"); apList.BackgroundTransparency=1; apList.BorderSizePixel=0; apList.Position=UDim2.new(0,0,0,20); apList.Size=UDim2.new(1,0,0,0); apList.AutomaticSize=Enum.AutomaticSize.Y; apList.Parent=apOuter; corner(apList,3)
    apPad=Instance.new("UIPadding"); apPad.PaddingTop=UDim.new(0,2); apPad.PaddingBottom=UDim.new(0,2); apPad.PaddingLeft=UDim.new(0,4); apPad.PaddingRight=UDim.new(0,4); apPad.Parent=apList
    Instance.new("UIListLayout",apList).SortOrder=Enum.SortOrder.LayoutOrder; apList:FindFirstChildOfClass("UIListLayout").Padding=UDim.new(0,2)

    apRows,stealLabels={},{}; rc=0
    -- Build RP_BUTTONS dynamically from Config.AdminPanelButtons
    local function buildRPButtons()
        local btns = {}
        local order = Config.AdminPanelOrder or AP_ALL_COMMANDS
        for _, cmd in ipairs(order) do
            if Config.AdminPanelButtons and Config.AdminPanelButtons[cmd] then
                table.insert(btns, {AP_COMMAND_EMOJIS[cmd] or "⚡", cmd})
            end
        end
        if #btns == 0 then btns = {{"🤸","ragdoll"},{"🔒","jail"},{"🚀","rocket"},{"🎈","balloon"}} end
        return btns
    end
    RP_BUTTONS = buildRPButtons()

    _G.refreshAdminPanelRows = function()
        for uid, row in pairs(apRows) do
            if row then row:Destroy() end
        end
        table.clear(apRows)
        table.clear(stealLabels)
        rc = 0
        RP_BUTTONS = buildRPButtons()
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= player then createAPRow(plr) end
        end
    end

    function createAPRow(plr)
        if not plr or plr==player then return end; if apRows[plr.UserId] and apRows[plr.UserId].Parent then return end
        rc=rc+1
        local actW = #RP_BUTTONS * 32
        local rowW = math.max(460, actW + 180)
        local row=Instance.new("Frame"); row.Name="Row_"..plr.UserId; row.BackgroundColor3=Color3.fromRGB(22,24,26); row.BackgroundTransparency=0.35; row.BorderSizePixel=0; row.Size=UDim2.new(0,rowW,0,54); row.ZIndex=5; row.ClipsDescendants=false; row.Parent=apList; corner(row,10)
        apRows[plr.UserId]=row

        local avatarH=Instance.new("Frame"); avatarH.BackgroundTransparency=1; avatarH.BorderSizePixel=0; avatarH.Size=UDim2.fromOffset(34,34); avatarH.Position=UDim2.fromOffset(8,10); avatarH.Parent=row; corner(avatarH,8)
        local avatar=Instance.new("ImageLabel"); avatar.BackgroundTransparency=1; avatar.Size=UDim2.fromScale(1,1); avatar.ZIndex=10; avatar.Parent=avatarH; corner(avatar,8)
        task.spawn(function() local ok,img=pcall(function() return Players:GetUserThumbnailAsync(plr.UserId,Enum.ThumbnailType.HeadShot,Enum.ThumbnailSize.Size48x48) end); if ok then avatar.Image=img end end)

        local txtW=-(actW+60)
        local nameLabel=Instance.new("TextLabel"); nameLabel.Size=UDim2.new(1,txtW,0,20); nameLabel.Position=UDim2.fromOffset(50,6); nameLabel.BackgroundTransparency=1; nameLabel.Text=plr.DisplayName; nameLabel.Font=Enum.Font.GothamBold; nameLabel.TextSize=14; nameLabel.TextColor3=Theme.Text; nameLabel.TextXAlignment=Enum.TextXAlignment.Left; nameLabel.ZIndex=10; nameLabel.Parent=row
        local userL=Instance.new("TextLabel"); userL.BackgroundTransparency=1; userL.Position=UDim2.fromOffset(50,23); userL.Size=UDim2.new(1,txtW,0,16); userL.TextXAlignment=Enum.TextXAlignment.Left; userL.Text="@"..plr.Name; userL.Font=Enum.Font.GothamMedium; userL.TextSize=10; userL.TextColor3=Theme.Dim; userL.ZIndex=10; userL.Parent=row
        local statusL=Instance.new("TextLabel"); statusL.BackgroundTransparency=1; statusL.Position=UDim2.fromOffset(50,37); statusL.Size=UDim2.new(1,txtW,0,14); statusL.TextXAlignment=Enum.TextXAlignment.Left; statusL.Text=""; statusL.Font=Enum.Font.GothamBold; statusL.TextSize=11; statusL.TextColor3=Theme.AccentLight; statusL.ZIndex=10; statusL.Parent=row
        stealLabels[plr.UserId]=statusL

        local actions=Instance.new("Frame"); actions.BackgroundTransparency=1; actions.AnchorPoint=Vector2.new(1,0.5); actions.Position=UDim2.new(1,-8,0.5,0); actions.Size=UDim2.fromOffset(actW+10,38); actions.ZIndex=12; actions.Parent=row
        local al=Instance.new("UIListLayout"); al.FillDirection=Enum.FillDirection.Horizontal; al.SortOrder=Enum.SortOrder.LayoutOrder; al.Padding=UDim.new(0,2); al.Parent=actions

        for i,b in ipairs(RP_BUTTONS) do
            local btn=Instance.new("TextButton"); btn.Size=UDim2.fromOffset(30,30); btn.BackgroundTransparency=1; btn.AutoButtonColor=false; btn.Text=b[1]; btn.TextSize=14; btn.LayoutOrder=i; btn.Parent=actions
            btn.BackgroundColor3=Theme.SoftButton; btn.Font=Enum.Font.GothamBold; btn.TextColor3=Color3.new(1,1,1); btn.ZIndex=13
            btn.MouseButton1Click:Connect(function()
                if isPlayerBlacklisted(plr) then
                    ShowNotification("BLOCKED", plr.DisplayName .. " is blacklisted")
                    return
                end
                if apIsOnCooldown(b[2]) then return end; pcall(runAdminCommand,plr,b[2])
            end)
        end

        local function updateBlacklistVisuals()
            local isBlacklisted = isPlayerBlacklisted(plr)
            if isBlacklisted then
                -- no more dimming: every row stays fully readable (the
                -- protection itself is enforced in runAdminCommand anyway)
                avatar.ImageTransparency = 0
                nameLabel.TextTransparency = 0
                userL.TextTransparency = 0
                statusL.TextTransparency = 0
                for _, child in ipairs(actions:GetChildren()) do
                    if child:IsA("TextButton") then
                        child.TextTransparency = 0.8
                    end
                end
            else
                avatar.ImageTransparency = 0
                nameLabel.TextTransparency = 0
                userL.TextTransparency = 0
                statusL.TextTransparency = 0
                for _, child in ipairs(actions:GetChildren()) do
                    if child:IsA("TextButton") then
                        child.TextTransparency = 0
                    end
                end
            end
        end

        updateBlacklistVisuals()

        -- Click on player row = fire ALL available admin commands at once (like Click to AP)
        row.InputBegan:Connect(function(input)
            if input.UserInputType ~= Enum.UserInputType.MouseButton1 then return end
            -- Ignore clicks on the action buttons themselves
            local mousePos = UIS:GetMouseLocation()
            if actions and actions.AbsolutePosition and actions.AbsoluteSize then
                local ax, ay = actions.AbsolutePosition.X, actions.AbsolutePosition.Y
                local aw, ah = actions.AbsoluteSize.X, actions.AbsoluteSize.Y
                if mousePos.X >= ax and mousePos.X <= ax+aw and mousePos.Y >= ay and mousePos.Y <= ay+ah then
                    return
                end
            end
            if isPlayerBlacklisted(plr) then
                ShowNotification("BLOCKED", plr.DisplayName .. " is blacklisted")
                return
            end
            task.spawn(function()
                local activeCmds = {}
                for _,cmd in ipairs(AP_ALL_COMMANDS) do
                    if not apIsOnCooldown(cmd) then
                        table.insert(activeCmds, cmd)
                    end
                end
                if #activeCmds == 0 then
                    ShowNotification("ADMIN PANEL", "All commands on cooldown")
                    return
                end
                -- Fire ALL at once with minimal delay
                for i, cmd in ipairs(activeCmds) do
                    task.spawn(function()
                        task.wait((i - 1) * 0.01)
                        runAdminCommand(plr, cmd)
                    end)
                end
                ShowNotification("ADMIN PANEL", "Fired " .. #activeCmds .. " cmds on " .. plr.DisplayName)
            end)
        end)

        -- Steal / base-owner status. Reads the cached Synchronizer + memoized
        -- base-owner set, so every row shares one scan instead of each rescanning.
        task.spawn(function() while row.Parent do task.wait(0.5)
            if not plr or not plr.Parent then break end
            local st=stealLabels[plr.UserId]; if not st then break end

            local stealOwner, stealPet = getStealingInfo(plr)
            local hubBadge, hubColor, joinerTag, hubBadges = getHubUserBadge(plr)
            local curBaseOwnerId = _G.__getCurrentBaseOwnerId()

            -- Hub badges take priority, shown right next to the name, side by
            -- side when a player matches several hubs (RichText: each badge has
            -- its own color, the name keeps its own)
            pcall(function()
                if hubBadges then
                    nameLabel.RichText = true
                    local parts = {}
                    for _, def in ipairs(hubBadges) do
                        local hex = string.format("#%02X%02X%02X",
                            math.floor(def.color.R * 255), math.floor(def.color.G * 255), math.floor(def.color.B * 255))
                        parts[#parts + 1] = '<font color="' .. hex .. '"><b>' .. def.badge .. '</b></font>'
                    end
                    nameLabel.Text = plr.DisplayName .. '  ' .. table.concat(parts, " ")
                elseif nameLabel.RichText then
                    nameLabel.RichText = false
                    nameLabel.Text = plr.DisplayName
                end
            end)

            -- LIVE SORT: stealers pinned to the TOP of the panel, then the
            -- base owner, then everyone else in their usual order.
            -- NOTE: the raw joiner-tag line is gone (the badges NEXT TO the
            -- name already show hub identity -- it was displayed twice).
            local isOwner = curBaseOwnerId and curBaseOwnerId == plr.UserId
            -- active admin effect read from the overhead tag (inversed, jail...)
            local fx = nil
            if joinerTag then
                local raw = joinerTag:lower()
                for _, kw in ipairs({ "invers", "jail", "jumpscare", "morph", "nightvision", "rocket", "tiny", "balloon" }) do
                    if raw:find(kw, 1, true) then
                        fx = kw:sub(1, 1):upper() .. kw:sub(2)
                        if fx == "Invers" then fx = "Inversed" end
                        break
                    end
                end
            end
            if stealPet then
                -- short & clean: just the brainrot name, no victim/tag clutter
                st.Text = "● Stealing : " .. tostring(stealPet)
                st.TextColor3 = Color3.fromRGB(255, 90, 90)
                row.LayoutOrder = 1
            elseif isOwner then
                -- Base Owner NEVER disappears; the admin effect is appended
                st.Text = fx and ("● Base Owner | " .. fx) or "● Base Owner"
                st.TextColor3 = Color3.fromRGB(90, 230, 120)
                row.LayoutOrder = 2
            elseif fx then
                st.Text = "● " .. fx
                st.TextColor3 = Color3.fromRGB(255, 190, 70)
                row.LayoutOrder = 10
            else
                st.Text = ""
                row.LayoutOrder = 10
            end

            -- Danger-tool name highlight
            pcall(function() local ht=getHeldTool(plr)
                if ht and DANGER_TOOLS[ht] then nameLabel.TextColor3=Color3.fromRGB(255,60,60) else nameLabel.TextColor3=Theme.Text end
            end)
        end end)
    end

    for _,plr in ipairs(Players:GetPlayers()) do if plr~=player then createAPRow(plr) end end
    Players.PlayerAdded:Connect(function(plr) task.defer(function() createAPRow(plr) end) end)
    Players.PlayerRemoving:Connect(function(plr) local row=apRows[plr.UserId]; if row then row:Destroy(); apRows[plr.UserId]=nil end; stealLabels[plr.UserId]=nil end)
end) -- END ADMIN PANEL UI SCOPE (LazyInit)

-- ============================================================
-- SIDEBAR NAV + TABS
-- ============================================================
function styleSidebarTabs(activeTab)
    for name, btn in pairs(tabButtons) do
        local active = name == activeTab
        btn.BackgroundColor3 = Theme.SidebarActive or Color3.fromRGB(55,55,65)
        btn.BackgroundTransparency = active and (UITransparency.SidebarTab or 0.12) or 0.82
        btn.TextColor3 = active and (Theme.SidebarActiveText or Theme.Accent or Theme.Text) or (Theme.SidebarInactiveText or Theme.Text)
        btn.Font = active and Enum.Font.GothamBold or Enum.Font.GothamMedium
        local ind = btn:FindFirstChild("ActiveInd")
        if ind then ind.Visible = active; ind.BackgroundColor3 = Theme.Accent or Color3.fromRGB(74,140,255) end
    end
end
_G.styleSidebarTabs = styleSidebarTabs

-- TOP HORIZONTAL TAB BAR: flat tab order matching the image layout
tabGroups={
    {label="MAIN",       tabs={"Auto TP","Priority"}},
    {label="VISUALS",    tabs={"ESP","UI"}},
    {label="ADDITIONAL", tabs={"Misc","Keybinds"}},
    {label="GLOBAL",     tabs={"Performance"}},
}
do
    -- Flat ordered list matching: Keybinds | Auto TP | ESP | UI | Misc | Priority | Performance
    local flatOrder={"Keybinds","Auto TP","ESP","UI","Misc","Priority","Performance"}
    -- Add any tabs from tabGroups not already in the order list
    for _,grp in ipairs(tabGroups) do
        for _,name in ipairs(grp.tabs) do
            local found=false; for _,fn in ipairs(flatOrder) do if fn==name then found=true; break end end
            if not found then table.insert(flatOrder,name) end
        end
    end
    for i,name in ipairs(flatOrder) do
        -- verify tab exists in tabGroups
        local exists=false
        for _,grp in ipairs(tabGroups) do for _,tn in ipairs(grp.tabs) do if tn==name then exists=true; break end end; if exists then break end end
        if not exists then continue end
        local b=Instance.new("TextButton")
        b.Size=UDim2.new(0,0,0,26)
        b.AutomaticSize=Enum.AutomaticSize.X
        b.LayoutOrder=i
        b.BackgroundColor3=Theme.SidebarActive or Color3.fromRGB(55,55,65)
        b.BackgroundTransparency=0.82
        b.Text=name
        b.TextColor3=Theme.SidebarInactiveText or Theme.Text
        b.Font=Enum.Font.GothamMedium
        b.TextSize=10
        b.AutoButtonColor=false
        b.Parent=sidebarNav
        corner(b,6)
        local bp=Instance.new("UIPadding"); bp.PaddingLeft=UDim.new(0,10); bp.PaddingRight=UDim.new(0,10); bp.Parent=b
        -- Bottom accent indicator (horizontal bar at bottom of tab)
        local ind=Instance.new("Frame")
        ind.Name="ActiveInd"
        ind.Size=UDim2.new(1,-6,0,2)
        ind.Position=UDim2.new(0,3,1,-2)
        ind.BackgroundColor3=Theme.Accent or Color3.fromRGB(74,140,255)
        ind.BorderSizePixel=0
        ind.Visible=false
        ind.Parent=b
        corner(ind,1)
        b.MouseEnter:Connect(function()
            if UI.CurrentTab~=name then tw(b,{TextColor3=Theme.Text, BackgroundTransparency=0.6},0.1) end
        end)
        b.MouseLeave:Connect(function()
            if UI.CurrentTab~=name then tw(b,{TextColor3=Theme.SidebarInactiveText or Theme.Text, BackgroundTransparency=0.82},0.1) end
        end)
        tabButtons[name]=b
    end
end

function makeToolDropdown(parent, labelText, options, selected, onChange)
    local container=Instance.new("Frame")
    container.Size=UDim2.new(1,-4,0,36)
    container.BackgroundTransparency=1
    container.Parent=parent

    local lbl=Instance.new("TextLabel")
    lbl.Size=UDim2.new(0.42,0,1,0)
    lbl.BackgroundTransparency=1
    lbl.Text=labelText
    lbl.TextColor3=Theme.Text
    lbl.Font=Enum.Font.GothamMedium
    lbl.TextSize=12
    lbl.TextXAlignment=Enum.TextXAlignment.Left
    lbl.Parent=container

    local current=selected or options[1]
    local dropBtn=Instance.new("TextButton")
    dropBtn.Size=UDim2.new(0.56,-4,0,26)
    dropBtn.Position=UDim2.new(0.44,0,0.5,-13)
    dropBtn.BackgroundColor3=Theme.InputBg
    dropBtn.BackgroundTransparency=UITransparency.Input
    dropBtn.Text=current
    dropBtn.TextColor3=Theme.Text
    dropBtn.Font=Enum.Font.GothamMedium
    dropBtn.TextSize=10
    dropBtn.AutoButtonColor=false
    dropBtn.Parent=container
    corner(dropBtn,8)
    stroke(dropBtn,Theme.Stroke,1,0.65)

    local menuFrame, closeConn = nil, nil
    local function closeMenu()
        if closeConn then closeConn:Disconnect(); closeConn=nil end
        if menuFrame then menuFrame:Destroy(); menuFrame=nil end
    end
    local function setSelected(val)
        current=val
        dropBtn.Text=val
    end
    local function openMenu()
        closeMenu()
        if not main then return end
        local abs=dropBtn.AbsolutePosition
        local w=dropBtn.AbsoluteSize.X
        menuFrame=Instance.new("Frame")
        menuFrame.Size=UDim2.fromOffset(w,#options*28+4)
        menuFrame.Position=UDim2.fromOffset(abs.X,abs.Y+dropBtn.AbsoluteSize.Y+2)
        menuFrame.BackgroundColor3=Theme.InputBg
        menuFrame.BorderSizePixel=0
        menuFrame.ZIndex=200
        menuFrame.Parent=gui
        corner(menuFrame,6)
        stroke(menuFrame,Theme.Stroke,1,0.35)
        for i,opt in ipairs(options) do
            local optBtn=Instance.new("TextButton")
            optBtn.Size=UDim2.new(1,-4,0,26)
            optBtn.Position=UDim2.new(0,2,0,(i-1)*28+2)
            optBtn.BackgroundColor3=opt==current and (Theme.SidebarActive or Color3.fromRGB(245,245,247)) or Theme.InputBg
            optBtn.Text=opt
            optBtn.TextColor3=opt==current and (Theme.SidebarActiveText or Color3.fromRGB(18,18,20)) or Theme.Text
            optBtn.Font=Enum.Font.GothamMedium
            optBtn.TextSize=9
            optBtn.AutoButtonColor=false
            optBtn.ZIndex=201
            optBtn.Parent=menuFrame
            corner(optBtn,4)
            optBtn.MouseButton1Click:Connect(function()
                setSelected(opt)
                if onChange then onChange(opt) end
                closeMenu()
            end)
        end
        closeConn=UIS.InputBegan:Connect(function(input)
            if input.UserInputType~=Enum.UserInputType.MouseButton1 and input.UserInputType~=Enum.UserInputType.Touch then return end
            local p=input.Position
            local inMenu=menuFrame and p.X>=menuFrame.AbsolutePosition.X and p.X<=menuFrame.AbsolutePosition.X+menuFrame.AbsoluteSize.X and p.Y>=menuFrame.AbsolutePosition.Y and p.Y<=menuFrame.AbsolutePosition.Y+menuFrame.AbsoluteSize.Y
            local inBtn=p.X>=abs.X and p.X<=abs.X+w and p.Y>=abs.Y and p.Y<=abs.Y+dropBtn.AbsoluteSize.Y
            if not inMenu and not inBtn then closeMenu() end
        end)
    end
    dropBtn.MouseButton1Click:Connect(function() if menuFrame then closeMenu() else openMenu() end end)
    return setSelected
end

-- Cache all animal names for autocomplete. Source per request:
-- ReplicatedStorage.Animations.Animals (animation instances are named after
-- each brainrot). Fallback: ReplicatedStorage.Models.Animals if Animations
-- isn't present in the place.
_allAnimalNames = nil
function getAllAnimalNames()
    if _allAnimalNames and #_allAnimalNames > 0 then return _allAnimalNames end
    _allAnimalNames = {}
    pcall(function()
        local source = ReplicatedStorage:FindFirstChild("Animations")
        source = source and source:FindFirstChild("Animals")
        if not source then
            -- fallback if Animations.Animals isn't present
            local models = ReplicatedStorage:FindFirstChild("Models")
            source = models and models:FindFirstChild("Animals")
        end
        if source then
            local seen = {}
            for _, child in ipairs(source:GetChildren()) do
                if not seen[child.Name] then
                    seen[child.Name] = true
                    table.insert(_allAnimalNames, child.Name)
                end
            end
            table.sort(_allAnimalNames, function(a, b) return a:lower() < b:lower() end)
        end
    end)
    return _allAnimalNames
end
task.spawn(getAllAnimalNames)

-- Drag state for priority reorder
_priDragState = {active = false, fromIndex = nil, ghostFrame = nil, overlay = nil}

function cleanupPriDrag()
    if _priDragState.ghostFrame then pcall(function() _priDragState.ghostFrame:Destroy() end); _priDragState.ghostFrame = nil end
    if _priDragState.overlay then pcall(function() _priDragState.overlay:Destroy() end); _priDragState.overlay = nil end
    _priDragState.active = false; _priDragState.fromIndex = nil
end

function makePriorityRow(index)
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,31); row.BackgroundColor3=Theme.Panel; row.BackgroundTransparency=UITransparency.Row; row.Parent=mainBody; corner(row,6); row.LayoutOrder=index

    -- Number label
    local num=Instance.new("TextLabel"); num.Size=UDim2.new(0,24,1,0); num.Position=UDim2.new(0,4,0,0); num.BackgroundTransparency=1; num.Text=tostring(index).."."; num.TextColor3=Theme.Dim; num.Font=Enum.Font.GothamBold; num.TextSize=10; num.TextXAlignment=Enum.TextXAlignment.Left; num.Parent=row

    local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-120,1,0); l.Position=UDim2.new(0,28,0,0); l.BackgroundTransparency=1; l.Text=priorityList[index]; l.TextColor3=Theme.Text; l.Font=Enum.Font.GothamMedium; l.TextSize=10; l.TextXAlignment=Enum.TextXAlignment.Left; l.TextTruncate=Enum.TextTruncate.AtEnd; l.Parent=row

    local up=Instance.new("TextButton"); up.Name="WhiteTextBtn"; up.Size=UDim2.new(0,26,0,22); up.Position=UDim2.new(1,-86,0.5,-11); up.BackgroundColor3=Theme.Accent; up.BackgroundTransparency=UITransparency.Toggle; up.Text="▲"; up.TextColor3=Color3.fromRGB(18,18,20); up.Font=Enum.Font.GothamBold; up.TextSize=10; up.Parent=row; corner(up,5)
    local dn=Instance.new("TextButton"); dn.Name="WhiteTextBtn"; dn.Size=UDim2.new(0,26,0,22); dn.Position=UDim2.new(1,-56,0.5,-11); dn.BackgroundColor3=Theme.Accent; dn.BackgroundTransparency=UITransparency.Toggle; dn.Text="▼"; dn.TextColor3=Color3.fromRGB(18,18,20); dn.Font=Enum.Font.GothamBold; dn.TextSize=10; dn.Parent=row; corner(dn,5)
    local del=Instance.new("TextButton"); del.Name="WhiteTextBtn"; del.Size=UDim2.new(0,26,0,22); del.Position=UDim2.new(1,-26,0.5,-11); del.BackgroundColor3=Theme.Red; del.BackgroundTransparency=UITransparency.Toggle; del.Text="X"; del.TextColor3=Color3.new(1,1,1); del.Font=Enum.Font.GothamBold; del.TextSize=10; del.Parent=row; corner(del,5)
    up.MouseButton1Click:Connect(function() if index>1 then priorityList[index],priorityList[index-1]=priorityList[index-1],priorityList[index]; Config.PriorityList=priorityList; saveConfig(); loadTab("Priority") end end)
    dn.MouseButton1Click:Connect(function() if index<#priorityList then priorityList[index],priorityList[index+1]=priorityList[index+1],priorityList[index]; Config.PriorityList=priorityList; saveConfig(); loadTab("Priority") end end)
    del.MouseButton1Click:Connect(function() local removedName=priorityList[index]; table.remove(priorityList,index); if not Config.RemovedFromPriority then Config.RemovedFromPriority={} end; local alreadyRemoved=false; for _,rn in ipairs(Config.RemovedFromPriority) do if rn==removedName then alreadyRemoved=true; break end end; if not alreadyRemoved then table.insert(Config.RemovedFromPriority,removedName) end; Config.PriorityList=priorityList; saveConfig(); loadTab("Priority") end)

    -- Drag & Drop
    local dragHandle = Instance.new("TextButton"); dragHandle.Size=UDim2.new(0,24,1,0); dragHandle.Position=UDim2.new(0,0,0,0); dragHandle.BackgroundTransparency=1; dragHandle.Text=""; dragHandle.ZIndex=9; dragHandle.Parent=row
    dragHandle.InputBegan:Connect(function(input)
        if input.UserInputType ~= Enum.UserInputType.MouseButton1 and input.UserInputType ~= Enum.UserInputType.Touch then return end
        cleanupPriDrag()
        _priDragState.active = true; _priDragState.fromIndex = index

        local ghost = Instance.new("Frame"); ghost.Size = UDim2.new(0, row.AbsoluteSize.X, 0, 31)
        ghost.BackgroundColor3 = Theme.Accent; ghost.BackgroundTransparency = 0.55; ghost.BorderSizePixel = 0; ghost.ZIndex = 100
        ghost.Parent = gui_sg; corner(ghost, 6)
        local gl = Instance.new("TextLabel"); gl.Size = UDim2.new(1,0,1,0); gl.BackgroundTransparency = 1; gl.Text = tostring(index)..". "..priorityList[index]; gl.TextColor3 = Color3.fromRGB(18,18,20); gl.Font = Enum.Font.GothamBold; gl.TextSize = 10; gl.Parent = ghost
        _priDragState.ghostFrame = ghost

        local moveConn, endConn
        moveConn = UIS.InputChanged:Connect(function(mi)
            if mi.UserInputType == Enum.UserInputType.MouseMovement or mi.UserInputType == Enum.UserInputType.Touch then
                ghost.Position = UDim2.new(0, mi.Position.X - ghost.AbsoluteSize.X/2, 0, mi.Position.Y - 15)
            end
        end)
        endConn = UIS.InputEnded:Connect(function(ei)
            if ei.UserInputType ~= Enum.UserInputType.MouseButton1 and ei.UserInputType ~= Enum.UserInputType.Touch then return end
            moveConn:Disconnect(); endConn:Disconnect()
            if not _priDragState.active then cleanupPriDrag(); return end

            -- Determine drop target index from Y position
            local dropY = ei.Position.Y
            local targetIndex = #priorityList
            for _, child in ipairs(mainBody:GetChildren()) do
                if child:IsA("Frame") and child.LayoutOrder and child.LayoutOrder >= 1 and child.LayoutOrder <= #priorityList then
                    local absY = child.AbsolutePosition.Y
                    local absH = child.AbsoluteSize.Y
                    if dropY < absY + absH / 2 then
                        targetIndex = child.LayoutOrder
                        break
                    end
                end
            end
            local fromIdx = _priDragState.fromIndex
            cleanupPriDrag()
            if fromIdx and targetIndex and fromIdx ~= targetIndex then
                local item = table.remove(priorityList, fromIdx)
                if targetIndex > fromIdx then targetIndex = targetIndex - 1 end
                targetIndex = math.clamp(targetIndex, 1, #priorityList + 1)
                table.insert(priorityList, targetIndex, item)
                Config.PriorityList = priorityList; saveConfig()
                loadTab("Priority")
            end
        end)
    end)
end

function makePriorityAddRow()
    local holder = Instance.new("Frame"); holder.Size=UDim2.new(1,-4,0,31); holder.BackgroundColor3=Theme.SoftAccent; holder.BackgroundTransparency=UITransparency.SoftRow; holder.ClipsDescendants=false; holder.Parent=mainBody; holder.ZIndex=20; corner(holder,6); holder.LayoutOrder = -2

    local box=Instance.new("TextBox"); box.Size=UDim2.new(1,-60,1,-6); box.Position=UDim2.new(0,6,0,3); box.BackgroundColor3=Theme.InputBg; box.BackgroundTransparency=UITransparency.Input; box.BorderSizePixel=0; box.Text=""; box.PlaceholderText="Enter pet name..."; box.TextColor3=Theme.Text; box.PlaceholderColor3=Theme.Dim; box.Font=Enum.Font.GothamMedium; box.TextSize=10; box.ClearTextOnFocus=false; box.Parent=holder; box.ZIndex=21; corner(box,4)

    local addBtn=Instance.new("TextButton"); addBtn.Name="WhiteTextBtn"; addBtn.Size=UDim2.new(0,44,0,25); addBtn.Position=UDim2.new(1,-50,0.5,-12.5); addBtn.BackgroundColor3=Theme.Accent; addBtn.BackgroundTransparency=UITransparency.Toggle; addBtn.Text="ADD"; addBtn.TextColor3=Color3.fromRGB(18,18,20); addBtn.Font=Enum.Font.GothamBlack; addBtn.TextSize=10; addBtn.AutoButtonColor=false; addBtn.Parent=holder; addBtn.ZIndex=21; corner(addBtn,5)

    -- Dropdown for autocomplete
    local dropdown = Instance.new("Frame"); dropdown.Name="PriorityDropdown"; dropdown.Size=UDim2.new(1,-60,0,0); dropdown.Position=UDim2.new(0,6,1,2)
    dropdown.BackgroundColor3=Theme.Background; dropdown.BackgroundTransparency=UITransparency.Panel; dropdown.BorderSizePixel=0; dropdown.ClipsDescendants=true; dropdown.Visible=false; dropdown.ZIndex=50; dropdown.Parent=holder; corner(dropdown,6)
    local ddStroke = Instance.new("UIStroke"); ddStroke.Color=Theme.AccentLight; ddStroke.Thickness=1; ddStroke.Parent=dropdown
    local ddScroll = Instance.new("ScrollingFrame"); ddScroll.Size=UDim2.new(1,0,1,0); ddScroll.BackgroundTransparency=1; ddScroll.BorderSizePixel=0; ddScroll.ScrollBarThickness=3; ddScroll.ScrollBarImageColor3=Theme.Accent; ddScroll.CanvasSize=UDim2.new(0,0,0,0); ddScroll.Active=true; ddScroll.ZIndex=51; ddScroll.Parent=dropdown
    local ddLayout = Instance.new("UIListLayout"); ddLayout.Padding=UDim.new(0,1); ddLayout.Parent=ddScroll
    ddLayout:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function() ddScroll.CanvasSize=UDim2.new(0,0,0,ddLayout.AbsoluteContentSize.Y) end)

    local function addName(name)
        local trimmed = name:match("^%s*(.-)%s*$")
        if not trimmed or trimmed == "" then return end
        local exists = false
        for _, pName in ipairs(priorityList) do if pName == trimmed then exists = true; break end end
        if not exists then
            table.insert(priorityList, trimmed)
            if Config.RemovedFromPriority then
                for i=#Config.RemovedFromPriority,1,-1 do
                    if Config.RemovedFromPriority[i] == trimmed then table.remove(Config.RemovedFromPriority, i) end
                end
            end
            Config.PriorityList=priorityList; saveConfig()
        end
        box.Text=""; dropdown.Visible=false; loadTab("Priority")
    end

    local function updateDropdown(query)
        for _, c in ipairs(ddScroll:GetChildren()) do if c:IsA("TextButton") then c:Destroy() end end
        if not query or query == "" then dropdown.Visible = false; return end
        local names = getAllAnimalNames()
        local q = query:lower()
        local matches = {}
        for _, name in ipairs(names) do
            if name:lower():find(q, 1, true) then
                local exists = false
                for _, pName in ipairs(priorityList) do if pName == name then exists = true; break end end
                if not exists then table.insert(matches, name); if #matches >= 8 then break end end
            end
        end
        if #matches == 0 then dropdown.Visible = false; return end
        local itemH = 24
        dropdown.Size = UDim2.new(1,-60,0,math.min(#matches,6)*itemH)
        dropdown.Visible = true
        for _, name in ipairs(matches) do
            local btn = Instance.new("TextButton"); btn.Size=UDim2.new(1,0,0,itemH); btn.BackgroundColor3=Theme.Row; btn.BackgroundTransparency=UITransparency.Dropdown; btn.Text=name; btn.TextColor3=Theme.Text; btn.Font=Enum.Font.GothamMedium; btn.TextSize=10; btn.AutoButtonColor=false; btn.ZIndex=52; btn.Parent=ddScroll
            btn.MouseEnter:Connect(function() btn.BackgroundColor3=Theme.RowHover end)
            btn.MouseLeave:Connect(function() btn.BackgroundColor3=Theme.Row end)
            btn.MouseButton1Click:Connect(function() addName(name) end)
        end
    end

    box:GetPropertyChangedSignal("Text"):Connect(function() updateDropdown(box.Text) end)
    box.FocusLost:Connect(function() task.delay(0.15, function() dropdown.Visible = false end) end)
    box.Focused:Connect(function() if box.Text ~= "" then updateDropdown(box.Text) end end)

    addBtn.MouseButton1Click:Connect(function() addName(box.Text) end)
end

function makePriorityRestoreRow()
    if not Config.RemovedFromPriority or #Config.RemovedFromPriority == 0 then return end
    local row=Instance.new("Frame"); row.Size=UDim2.new(1,-4,0,31); row.BackgroundColor3=Theme.SoftAccent; row.BackgroundTransparency=UITransparency.SoftRow; row.Parent=mainBody; corner(row,6); row.LayoutOrder = -1
    local l=Instance.new("TextLabel"); l.Size=UDim2.new(1,-96,1,0); l.Position=UDim2.new(0,8,0,0); l.BackgroundTransparency=1; l.Text=tostring(#Config.RemovedFromPriority).." ignored pet(s) available"; l.TextColor3=Theme.Dim; l.Font=Enum.Font.GothamMedium; l.TextSize=10; l.TextXAlignment=Enum.TextXAlignment.Left; l.Parent=row
    local restBtn=Instance.new("TextButton"); restBtn.Name="WhiteTextBtn"; restBtn.Size=UDim2.new(0,74,0,25); restBtn.Position=UDim2.new(1,-80,0.5,-12.5); restBtn.BackgroundColor3=Theme.Green; restBtn.BackgroundTransparency=UITransparency.Toggle; restBtn.Text="RESTORE"; restBtn.TextColor3=Color3.fromRGB(18,18,20); restBtn.Font=Enum.Font.GothamBlack; restBtn.TextSize=10; restBtn.AutoButtonColor=false; restBtn.Parent=row; corner(restBtn,5)
    restBtn.MouseButton1Click:Connect(function()
        for _, name in ipairs(Config.RemovedFromPriority) do
            local exists = false
            for _, pName in ipairs(priorityList) do
                if pName == name then exists = true; break end
            end
            if not exists then table.insert(priorityList, name) end
        end
        Config.RemovedFromPriority = {}
        Config.PriorityList = priorityList
        saveConfig()
        loadTab("Priority")
    end)
end

function loadTab(tabName)
    UI.CurrentTab=tabName; clearBody(mainBody)
    pcall(function() mainBody:FindFirstChildOfClass("UIListLayout").SortOrder = Enum.SortOrder.LayoutOrder end)
    styleSidebarTabs(tabName)

    if tabName=="Keybinds" then
        -- NO Steal Speed keybind, NO Unwalk
        for _,name in ipairs({"Kick","Clone","Drop Brainrot","Invisible Steal","Proximity","Carpet Boost","Open Menu"}) do makeKeybindRow(mainBody,name) end
        makeKeybindRow(mainBody,"Manual TP","Teleport")
        for _,name in ipairs({"Ragdoll Self","Float","Reset","Auto Buy","Click to AP","WalkSpeed"}) do makeKeybindRow(mainBody,name) end

    elseif tabName=="Auto TP" then
        makeMainToggle(mainBody,"Auto TP Priority Mode",Config.AutoTPPriority,function(on)
            Config.AutoTPPriority=on
            if on then
                if BoundToggles["Auto TP Highest Gen"] then BoundToggles["Auto TP Highest Gen"](false, false) end
                if BoundToggles["Auto TP Highest Value"] then BoundToggles["Auto TP Highest Value"](false, false) end
                Config.AutoTPHighestGen = false
                Config.AutoTPHighestValue = false
            end
            saveConfig()
        end)
        makeMainToggle(mainBody,"Auto TP Highest Gen",Config.AutoTPHighestGen,function(on)
            Config.AutoTPHighestGen=on
            if on then
                if BoundToggles["Auto TP Priority Mode"] then BoundToggles["Auto TP Priority Mode"](false, false) end
                if BoundToggles["Auto TP Highest Value"] then BoundToggles["Auto TP Highest Value"](false, false) end
                Config.AutoTPPriority = false
                Config.AutoTPHighestValue = false
            end
            saveConfig()
        end)
        makeMainToggle(mainBody,"Auto TP Highest Value",Config.AutoTPHighestValue,function(on)
            Config.AutoTPHighestValue=on
            if on then
                if BoundToggles["Auto TP Priority Mode"] then BoundToggles["Auto TP Priority Mode"](false, false) end
                if BoundToggles["Auto TP Highest Gen"] then BoundToggles["Auto TP Highest Gen"](false, false) end
                Config.AutoTPPriority = false
                Config.AutoTPHighestGen = false
            end
            saveConfig()
        end)
        makeMainToggle(mainBody,"TP on Load",Config.TpSettings.TpOnLoad,function(on) Config.TpSettings.TpOnLoad=on; saveConfig() end)
        makeMainSliderWithInput(mainBody, "TP on Load Delay", 0, 30, Config.TpSettings.TpOnLoadDelay or 3, function(v)
    Config.TpSettings.TpOnLoadDelay = v
    saveConfig()
end, "s")
        
        local toolOptions={"Flying Carpet","Cupid's Wings","Santa's Sleigh","Witch's Broom","Waverider"}
        makeToolDropdown(mainBody,"Teleport Tool",toolOptions,Config.TpSettings.Tool or "Flying Carpet",function(tn)
            Config.TpSettings.Tool=tn; saveConfig(); ShowNotification("TP TOOL",tn)
        end)
        makeMainToggle(mainBody,"Grabble TP",Config.TpSettings.GrabbleTP,function(on)
            Config.TpSettings.GrabbleTP=on
            saveConfig()
            -- Grabble TP IS the Mynxx Clone-TP engine. Sync auto-steal loop state.
            if _G.MynxxAutoSteal then
                pcall(_G.MynxxAutoSteal, on and (Config.AutoStealEnabled or false))
            end
        end)
        makeMainToggle(mainBody,"Carpet to Brainrot",Config.TpSettings.BrainrotCarpet,function(on) Config.TpSettings.BrainrotCarpet=on; saveConfig() end)

        makeMainButton(mainBody, "Tp Speed", function()
            if tpSpeedSettingsPanel.Visible then closeAnim(tpSpeedSettingsPanel) else openAnim(tpSpeedSettingsPanel) end
        end, Theme.Panel)
        makeMainTextBox(mainBody,"Min Gen for Auto TP",Config.TpSettings.MinGenForTp,"e.g. 50k, 1m, 10b",function(v)
            Config.TpSettings.MinGenForTp = v
            saveConfig()
            ShowNotification("MIN GEN TP", v == "" and "No minimum" or "Min: " .. v)
        end)

    elseif tabName=="ESP" then

        makeMainToggle(mainBody,"Player ESP",playerESPEnabled,function(on) playerESPEnabled=on; Config.PlayerESP=on; saveConfig(); if on then task.spawn(function() for _,pl in ipairs(Players:GetPlayers()) do if pl~=LocalPlayer then pcall(createOrRefreshPlayerESP,pl) end end end) else clearPlayerESP() end end)
        makeMainToggle(mainBody,"Brainrot ESP",brainrotESPEnabled,function(on) brainrotESPEnabled=on; Config.BrainrotESP=on; saveConfig(); if on then task.spawn(function() pcall(_G.refreshBrainrotESP) end) else _G.clearBrainrotESP() end end)
        makeMainToggle(mainBody,"TP Trail",Config.TpTrail ~= false,function(on) Config.TpTrail=on; saveConfig() end) -- ligne de trajet affichée pendant le TP
        makeSyncMainToggle(mainBody,"Timer ESP","Timer ESP",function(on)
            timerESPEnabled=on
            Config.TimerESP=on
            saveConfig()
            if on then task.spawn(function() pcall(refreshTimerESP) end) else clearTimerESP() end
        end)
        makeMainToggle(mainBody,"Subspace Mine ESP",subspaceMineESPEnabled,function(on) subspaceMineESPEnabled=on; Config.SubspaceMineESP=on; saveConfig() end)
        makeMainToggle(mainBody,"Base Owner ESP",Config.BaseOwnerESP,function(on) if _G.setBaseOwnerESP then _G.setBaseOwnerESP(on) end end)
        makeMainToggle(mainBody,"Line To Base",Config.LineToBase,function(on)
            Config.LineToBase=on; saveConfig()
            if on then if _G.createPlotBeam then pcall(_G.createPlotBeam) end
            else if _G.resetPlotBeam then pcall(_G.resetPlotBeam) end end
        end)
        makeMainToggle(mainBody,"Line To Best Brainrot",Config.LineToBrainrot or false,function(on)
            Config.LineToBrainrot=on; saveConfig()
            if on then -- beam auto-creates via heartbeat
            else if _G.resetBrainrotBeam then pcall(_G.resetBrainrotBeam) end end
        end)
        makeMainToggle(mainBody,"Brainrot Notification",Config.BrainrotAlert ~= false,function(on)
            Config.BrainrotAlert=on; saveConfig()
        end)

    elseif tabName=="UI" then
        for _,name in ipairs({"Invisible Steal Panel","Admin Command Panel","Command Cooldowns","Actions","Steal Panel","Steal Target","Grief Detector"}) do
            makeMainToggle(mainBody,name,panels[name].Visible,function(on)
                Config.Visibilities[name]=on
                saveConfig()
                if on then openAnim(panels[name]) else closeAnim(panels[name]) end
            end)
        end
        makeMainToggle(mainBody,"Clear Error Popups",Config.CleanErrorGUIs,function(on) Config.CleanErrorGUIs=on; saveConfig() end)
        makeMainToggle(mainBody,"Auto Close Main UI on Execute",Config.AutoCloseOnExec,function(on) Config.AutoCloseOnExec=on; saveConfig() end)
        makeMainToggle(mainBody,"Admin Panel UI",Config.AdminPanelUI ~= nil and Config.AdminPanelUI or true,function(on)
            Config.AdminPanelUI = on
            saveConfig()
            if apGui then apGui.Enabled = on end
        end)
        makeMainToggle(mainBody,"3D Pet Viewer",Config.Steal3DViewer ~= false,function(on) Config.Steal3DViewer=on; saveConfig() end)
        local shareRow = Instance.new("Frame")
        shareRow.Size = UDim2.new(1, -4, 0, 31)
        shareRow.BackgroundTransparency = 1
        shareRow.Parent = mainBody
        
        local shareBox = Instance.new("TextBox")
        shareBox.Size = UDim2.new(1, -12, 1, -6)
        shareBox.Position = UDim2.new(0, 6, 0, 3)
        shareBox.BackgroundColor3 = Theme.InputBg
        shareBox.BorderSizePixel = 0
        shareBox.Text = ""
        shareBox.PlaceholderText = "Paste config here to import, or click Export..."
        shareBox.TextColor3 = Theme.Text
        shareBox.Font = Enum.Font.GothamMedium
        shareBox.TextSize = 9
        shareBox.ClearTextOnFocus = false
        shareBox.Parent = shareRow
        corner(shareBox, 4)

        local btnRow = Instance.new("Frame")
        btnRow.Size = UDim2.new(1, -4, 0, 30)
        btnRow.BackgroundTransparency = 1
        btnRow.Parent = mainBody
        
        local expBtn = Instance.new("TextButton")
        expBtn.Size = UDim2.new(0.5, -3, 1, 0)
        expBtn.Position = UDim2.new(0, 0, 0, 0)
        expBtn.BackgroundColor3 = Theme.Row
        expBtn.BackgroundTransparency = UITransparency.MainButton
        expBtn.Text = "Export Config"
        expBtn.TextColor3 = Theme.Text
        expBtn.Font = Enum.Font.GothamBold
        expBtn.TextSize = 11
        expBtn.AutoButtonColor = false
        expBtn.Parent = btnRow
        corner(expBtn, 6)
        stroke(expBtn, Theme.Stroke, 1, 0.55)
        expBtn.MouseEnter:Connect(function() tw(expBtn, {BackgroundColor3 = Theme.RowHover}, 0.12) end)
        expBtn.MouseLeave:Connect(function() tw(expBtn, {BackgroundColor3 = Theme.Row}, 0.12) end)
        expBtn.MouseButton1Click:Connect(function()
            local str = _G.exportConfig()
            if str then
                shareBox.Text = str
            end
        end)
        
        local impBtn = Instance.new("TextButton")
        impBtn.Size = UDim2.new(0.5, -3, 1, 0)
        impBtn.Position = UDim2.new(0.5, 3, 0, 0)
        impBtn.BackgroundColor3 = Theme.Row
        impBtn.BackgroundTransparency = UITransparency.MainButton
        impBtn.Text = "Import Config"
        impBtn.TextColor3 = Theme.Text
        impBtn.Font = Enum.Font.GothamBold
        impBtn.TextSize = 11
        impBtn.AutoButtonColor = false
        impBtn.Parent = btnRow
        corner(impBtn, 6)
        stroke(impBtn, Theme.Stroke, 1, 0.55)
        impBtn.MouseEnter:Connect(function() tw(impBtn, {BackgroundColor3 = Theme.RowHover}, 0.12) end)
        impBtn.MouseLeave:Connect(function() tw(impBtn, {BackgroundColor3 = Theme.Row}, 0.12) end)
        impBtn.MouseButton1Click:Connect(function()
            _G.importConfig(shareBox.Text)
        end)

        -- White Mode / Dark Mode buttons removed â€” dark mode only
        -- (Theme is always dark)

        local rb=makeMainButton(mainBody,"Reset UI",function()
            main.Position=UDim2.new(0.5,-260,0.5,-220); panels["Invisible Steal Panel"].Position=UDim2.new(0,80,0.5,-220)
            panels["Admin Command Panel"].Position=UDim2.new(0.5,85,1,-340); panels["Command Cooldowns"].Position=UDim2.new(0.5,245,1,-390)
            panels["Actions"].Position=UDim2.new(0.5,505,1,-415); panels["Steal Panel"].Position=UDim2.new(1,-300,1,-420)
            panels["Steal Target"].Position=UDim2.new(1,-290,0,85); actionSettingsPanel.Position=UDim2.new(0.5,745,1,-400)
            bottomBar.Position=UDim2.new(0.5,0,1,-125); bottomBar.AnchorPoint=Vector2.new(0.5,0); Config.positions={}; saveConfig()
        end)
        local lockBtn; lockBtn=makeMainButton(mainBody,UI.Locked and "Locked" or "Unlocked",function() UI.Locked=not UI.Locked; Config.locked=UI.Locked; saveConfig(); lockBtn.Text=UI.Locked and "Locked" or "Unlocked"; if _G.updateResizeGrips then _G.updateResizeGrips() end end,Theme.SoftButton)
        
        makeMainToggle(mainBody,"Priority Sound Alert",Config.PrioritySoundAlert,function(on) Config.PrioritySoundAlert=on; saveConfig() end)
        makeMainTextBox(mainBody,"Custom Sound ID",Config.PrioritySoundID or "e.g. 123456789","e.g. 123456789",function(v) Config.PrioritySoundID=v; saveConfig() end)

    elseif tabName=="Misc" then
        -- NO Unwalk, NO Steal Speed, NO FPS Boost (moved to Performance)
        makeMainToggle(mainBody,"Instant Clone",true)
        -- Desync removed
        makeMainToggle(mainBody,"Auto Invisible During Steal",Config.AutoInvisDuringSteal,function(on) _G.AutoInvisDuringSteal=on; Config.AutoInvisDuringSteal=on; saveConfig() end)
        makeMainToggle(mainBody, "Anti Body Swap", _G.antiSwapEnabled or false, function(on)
    _G.setAntiSwapState(on)
end)
        makeMainToggle(mainBody,"Auto Unlock During Steal",Config.AutoUnlockOnSteal,function(on) Config.AutoUnlockOnSteal=on; saveConfig() end)
        makeSyncMainToggle(mainBody,"Anti Ragdoll","Anti Ragdoll",function(on) if on then startAntiRagdoll() else stopAntiRagdoll() end end)
        makeSyncMainToggle(mainBody,"Auto Reset Balloon","Auto Reset Balloon",function(on) Config.AutoResetBalloon=on; saveConfig() end)
        makeSyncMainToggle(mainBody,"Infinite Jump","Infinite Jump",function(on) setInfiniteJump(on) end)
        makeSyncMainToggle(mainBody,"Float","Float",function(on) if _G.toggleFloat and on ~= (FloatState and FloatState.active) then _G.toggleFloat() end end)
        makeSyncMainToggle(mainBody,"Auto Kick On Steal","Auto Kick",function(on) Config.AutoKickOnSteal=on; saveConfig() end)
        makeMainToggle(mainBody,"Kick to Private Server",Config.KickToPrivateServer,function(on) Config.KickToPrivateServer=on; saveConfig() end)
        makeMainTextBox(mainBody,"Private Server Code",PrivateServerCode,"e.g. ABC123XYZ...",function(v) PrivateServerCode=v; savePSCode() end)
        makeSyncMainToggle(mainBody,"Unwalk","Unwalk",function(on) if _G.setUnwalk then _G.setUnwalk(on) end end)
        makeSyncMainToggle(mainBody,"Invisible Steal","Invisible Steal",function(on) if _G.toggleInvisibleSteal then pcall(_G.toggleInvisibleSteal) end end)
        makeSyncMainToggle(mainBody,"Carpet Speed","Carpet Speed",function(on) setCarpetSpeed(on) end)
        makeSyncMainToggle(mainBody,"Click to AP","ClickToAP",function(on) Config.ClickToAP=on; saveConfig() end)

        -- Helper: create a config panel with toggles + ordering for AP commands
        local function makeAPConfigPanel(panelKey, titleText, configTable, orderKey, onToggleCb)
            if panels[panelKey] then
                panels[panelKey].Visible = not panels[panelKey].Visible
                return
            end

            -- Initialize order from config or default
            if not Config[orderKey] then
                Config[orderKey] = {}
                for i, cmd in ipairs(AP_ALL_COMMANDS) do Config[orderKey][i] = cmd end
            end
            local cmdOrder = Config[orderKey]

            local cfgPanel = Instance.new("Frame")
            cfgPanel.Name = panelKey
            cfgPanel.Size = UDim2.fromOffset(240, 0)
            cfgPanel.AutomaticSize = Enum.AutomaticSize.Y
            cfgPanel.Position = panelKey == "AdminPanelCmds" and UDim2.new(0.5, 200, 0.5, -200) or UDim2.new(0.5, 200, 0.5, -50)
            cfgPanel.BackgroundColor3 = Theme.Background
            cfgPanel.BackgroundTransparency = UITransparency.ConfigPanel
            cfgPanel.BorderSizePixel = 0
            cfgPanel.ZIndex = 100 -- High ZIndex to stay on top
            cfgPanel.Parent = gui_sg -- Parent to ScreenGui to act as a floating popup
            panels[panelKey] = cfgPanel
            corner(cfgPanel, 10)
            stroke(cfgPanel, Theme.Accent, 1.5, 0.3)
            makeDraggable(cfgPanel, cfgPanel)

            local cfgTitleLbl = Instance.new("TextLabel")
            cfgTitleLbl.Size = UDim2.new(1, 0, 0, 28)
            cfgTitleLbl.BackgroundColor3 = Theme.Panel
            cfgTitleLbl.BackgroundTransparency = 0.3
            cfgTitleLbl.Text = titleText
            cfgTitleLbl.TextColor3 = Theme.Text
            cfgTitleLbl.Font = Enum.Font.GothamBold
            cfgTitleLbl.TextSize = 12
            cfgTitleLbl.ZIndex = 101 -- Fix: Render on top of panel
            cfgTitleLbl.Parent = cfgPanel
            corner(cfgTitleLbl, 8)

            local cfgBody = Instance.new("Frame")
            cfgBody.Size = UDim2.new(1, -8, 0, 0)
            cfgBody.AutomaticSize = Enum.AutomaticSize.Y
            cfgBody.Position = UDim2.fromOffset(4, 32)
            cfgBody.BackgroundTransparency = 1
            cfgBody.ZIndex = 101 -- Fix: Render on top of panel
            cfgBody.Parent = cfgPanel
            Instance.new("UIListLayout", cfgBody).SortOrder = Enum.SortOrder.LayoutOrder
            cfgBody:FindFirstChildOfClass("UIListLayout").Padding = UDim.new(0, 2)

            local rowMap = {} -- cmd -> row frame
            local numMap = {} -- cmd -> number label

            local function refreshNumbers()
                for i, cmd in ipairs(cmdOrder) do
                    if rowMap[cmd] then rowMap[cmd].LayoutOrder = i end
                    if numMap[cmd] then numMap[cmd].Text = tostring(i) end
                end
            end

            local function swapOrder(idx1, idx2)
                if idx1 < 1 or idx2 < 1 or idx1 > #cmdOrder or idx2 > #cmdOrder then return end
                cmdOrder[idx1], cmdOrder[idx2] = cmdOrder[idx2], cmdOrder[idx1]
                Config[orderKey] = cmdOrder
                saveConfig()
                refreshNumbers()
            end

            for idx, cmd in ipairs(cmdOrder) do
                local emoji = AP_COMMAND_EMOJIS[cmd] or "⚡"
                local isOn = configTable[cmd] == true
                local row = Instance.new("Frame")
                row.Size = UDim2.new(1, 0, 0, 28)
                row.BackgroundColor3 = Theme.Panel
                row.BackgroundTransparency = UITransparency.RowAlt
                row.LayoutOrder = idx
                row.ZIndex = 102 -- Fix: Render on top of body
                row.Parent = cfgBody
                corner(row, 5)
                rowMap[cmd] = row

                -- Number label
                local numLbl = Instance.new("TextLabel")
                numLbl.Size = UDim2.fromOffset(16, 28)
                numLbl.Position = UDim2.fromOffset(2, 0)
                numLbl.BackgroundTransparency = 1
                numLbl.Text = tostring(idx)
                numLbl.TextColor3 = Theme.Dim
                numLbl.Font = Enum.Font.GothamBold
                numLbl.TextSize = 10
                numLbl.ZIndex = 103
                numLbl.Parent = row
                numMap[cmd] = numLbl

                -- Up arrow
                local upBtn = Instance.new("TextButton")
                upBtn.Size = UDim2.fromOffset(16, 13)
                upBtn.Position = UDim2.fromOffset(18, 1)
                upBtn.BackgroundTransparency = 1
                upBtn.Text = "▲"
                upBtn.TextColor3 = Theme.AccentLight
                upBtn.Font = Enum.Font.GothamBold
                upBtn.TextSize = 8
                upBtn.AutoButtonColor = false
                upBtn.ZIndex = 103
                upBtn.Parent = row

                -- Down arrow
                local dnBtn = Instance.new("TextButton")
                dnBtn.Size = UDim2.fromOffset(16, 13)
                dnBtn.Position = UDim2.fromOffset(18, 14)
                dnBtn.BackgroundTransparency = 1
                dnBtn.Text = "▼"
                dnBtn.TextColor3 = Theme.AccentLight
                dnBtn.Font = Enum.Font.GothamBold
                dnBtn.TextSize = 8
                dnBtn.AutoButtonColor = false
                dnBtn.ZIndex = 103
                dnBtn.Parent = row

                upBtn.MouseButton1Click:Connect(function()
                    local curIdx
                    for i, c in ipairs(cmdOrder) do if c == cmd then curIdx = i; break end end
                    if curIdx and curIdx > 1 then swapOrder(curIdx, curIdx - 1) end
                end)
                dnBtn.MouseButton1Click:Connect(function()
                    local curIdx
                    for i, c in ipairs(cmdOrder) do if c == cmd then curIdx = i; break end end
                    if curIdx and curIdx < #cmdOrder then swapOrder(curIdx, curIdx + 1) end
                end)

                -- Command label
                local lbl = Instance.new("TextLabel")
                lbl.Size = UDim2.new(1, -90, 1, 0)
                lbl.Position = UDim2.fromOffset(36, 0)
                lbl.BackgroundTransparency = 1
                lbl.Text = emoji .. " " .. cmd
                lbl.TextColor3 = Theme.Text
                lbl.Font = Enum.Font.GothamMedium
                lbl.TextSize = 11
                lbl.TextXAlignment = Enum.TextXAlignment.Left
                lbl.ZIndex = 103
                lbl.Parent = row

                -- Toggle button
                local btn = Instance.new("TextButton")
                btn.Size = UDim2.fromOffset(36, 18)
                btn.Position = UDim2.new(1, -42, 0.5, -9)
                btn.BackgroundColor3 = isOn and Theme.Green or Theme.ToggleOff
                btn.Text = isOn and "ON" or "OFF"
                btn.TextColor3 = Color3.new(1, 1, 1)
                btn.Font = Enum.Font.GothamBold
                btn.TextSize = 9
                btn.AutoButtonColor = false
                btn.ZIndex = 103
                btn.Parent = row
                corner(btn, 5)

                btn.MouseButton1Click:Connect(function()
                    isOn = not isOn
                    configTable[cmd] = isOn
                    saveConfig()
                    btn.BackgroundColor3 = isOn and Theme.Green or Theme.ToggleOff
                    btn.Text = isOn and "ON" or "OFF"
                    if onToggleCb then onToggleCb(cmd, isOn) end
                end)
            end

            if panelKey == "ClickToAPCmds" then
                local function setZIndex(inst)
                    if inst:IsA("GuiObject") then
                        inst.ZIndex = 103
                    end
                    for _, child in ipairs(inst:GetChildren()) do
                        setZIndex(child)
                    end
                end

                local sep = Instance.new("Frame")
                sep.Size = UDim2.new(1, -8, 0, 1)
                sep.Position = UDim2.new(0, 4, 0, 0)
                sep.BackgroundColor3 = Theme.Stroke
                sep.BackgroundTransparency = 0.2
                sep.LayoutOrder = 1000
                sep.ZIndex = 103
                sep.Parent = cfgBody

                local subHeader = Instance.new("TextLabel")
                subHeader.Size = UDim2.new(1, 0, 0, 24)
                subHeader.BackgroundTransparency = 1
                subHeader.Text = "Settings"
                subHeader.TextColor3 = Theme.Accent
                subHeader.Font = Enum.Font.GothamBold
                subHeader.TextSize = 10
                subHeader.LayoutOrder = 1001
                subHeader.ZIndex = 103
                subHeader.Parent = cfgBody

                local sliderWrapper = Instance.new("Frame")
                sliderWrapper.Size = UDim2.new(1, 0, 0, 50)
                sliderWrapper.BackgroundTransparency = 1
                sliderWrapper.LayoutOrder = 1002
                sliderWrapper.Parent = cfgBody

                local sliderObj = makeQuickSlider(sliderWrapper, "Click Radius", 1, 50, Config.ClickToAPRadius or 8, function(v)
                    Config.ClickToAPRadius = v
                    saveConfig()
                end, " studs")
                setZIndex(sliderWrapper)
            end

            if panelKey == "SpamBaseOwnerCmds" then
                local function setZIndex(inst)
                    if inst:IsA("GuiObject") then
                        inst.ZIndex = 103
                    end
                    for _, child in ipairs(inst:GetChildren()) do
                        setZIndex(child)
                    end
                end

                local sep = Instance.new("Frame")
                sep.Size = UDim2.new(1, -8, 0, 1)
                sep.Position = UDim2.new(0, 4, 0, 0)
                sep.BackgroundColor3 = Theme.Stroke
                sep.BackgroundTransparency = 0.2
                sep.LayoutOrder = 1000
                sep.ZIndex = 103
                sep.Parent = cfgBody

                local subHeader = Instance.new("TextLabel")
                subHeader.Size = UDim2.new(1, 0, 0, 24)
                subHeader.BackgroundTransparency = 1
                subHeader.Text = "Settings"
                subHeader.TextColor3 = Theme.Accent
                subHeader.Font = Enum.Font.GothamBold
                subHeader.TextSize = 10
                subHeader.LayoutOrder = 1001
                subHeader.ZIndex = 103
                subHeader.Parent = cfgBody

                local row = Instance.new("Frame")
                row.Size = UDim2.new(1, 0, 0, 34)
                row.BackgroundTransparency = 1
                row.LayoutOrder = 1002
                row.Parent = cfgBody

                local toggleFunc = makeSyncStateRow(row, "Single Command:", "SpamBaseOwnerSingleCommand", function(on)
                    Config.SpamBaseOwnerSingleCommand = on
                    saveConfig()
                end)
                setZIndex(row)
            end

            local pad = Instance.new("Frame")
            pad.Size = UDim2.new(1, 0, 0, 6)
            pad.BackgroundTransparency = 1
            pad.LayoutOrder = 1004
            pad.Parent = cfgBody
        end

        -- Admin Panel Commands button (controls which buttons show per player in admin panel)
        makeMainButton(mainBody, "Admin Panel Commands", function()
            makeAPConfigPanel("AdminPanelCmds", "Admin Panel Commands", Config.AdminPanelButtons, "AdminPanelOrder", function()
                if _G.refreshAdminPanelRows then _G.refreshAdminPanelRows() end
            end)
        end)

        -- Click to AP Commands button (controls which commands fire when you click a player)
        makeMainButton(mainBody, "Click to AP Commands", function()
            makeAPConfigPanel("ClickToAPCmds", "Click to AP Commands", Config.ClickToAPCommands, "ClickToAPOrder")
        end)

        -- Spam Base Owner Commands button
        makeMainButton(mainBody, "Spam Base Owner Commands", function()
            makeAPConfigPanel("SpamBaseOwnerCmds", "Spam Base Owner Commands", Config.SpamBaseOwnerCommands, "SpamBaseOwnerOrder")
        end)
        makeSyncMainToggle(mainBody,"Anti-Bee & Anti-Disco","AntiBeeDisco",function(on)
            Config.AntiBeeDisco = on
            saveConfig()
            if on then
                if _G.ANTI_BEE_DISCO and _G.ANTI_BEE_DISCO.Enable then
                    _G.ANTI_BEE_DISCO.Enable()
                end
            else
                if _G.ANTI_BEE_DISCO and _G.ANTI_BEE_DISCO.Disable then
                    _G.ANTI_BEE_DISCO.Disable()
                end
            end
        end)
        makeMainTextBox(mainBody,"Min Gen for Nearest Grab",Config.TpSettings.MinGenForGrab,"e.g. 10k, 500k, 1m",function(v)
            Config.TpSettings.MinGenForGrab = v
            saveConfig()
            ShowNotification("MIN GEN GRAB", v == "" and "No minimum" or "Min: " .. v)
        end)

    elseif tabName=="Priority" then
        makePriorityAddRow(); makePriorityRestoreRow(); for i=1,#priorityList do makePriorityRow(i) end

    elseif tabName=="Performance" then
        makeSyncMainToggle(mainBody,"Dark Theme","Dark Theme",function(on) setDarkTheme(on) end)
        makeQuickSlider(mainBody,"Dark Intensity",0,100,Config.DarkThemeIntensity or 50,function(v) setDarkThemeIntensity(v) end)
        makeSyncMainToggle(mainBody,"FPS Boost (normal)","FPS Boost (normal)",function(on) setFPSBoost(on) end)
        makeSyncMainToggle(mainBody,"FPS Boost Ultra","FPSBoostUltra",function(on) setFPSBoostUltra(on) end)
        makeSyncMainToggle(mainBody,"Kashi Optimizer","KashiOptimizer",function(on) setKashiOptimizer(on) end)
        makeSyncMainToggle(mainBody,"Xray","XRay",function(on) setXRay(on) end)
        makeQuickSlider(mainBody,"FOV",50,120,Config.FOV or 70,function(v) Config.FOV=v; saveConfig(); pcall(function() Workspace.CurrentCamera.FieldOfView=v end) end)

        -- ===== MENU COLOR ===============================================
        local secC=Instance.new("TextLabel"); secC.Size=UDim2.new(1,-4,0,22); secC.BackgroundTransparency=1; secC.Text="MENU COLOR"; secC.TextColor3=Theme.Dim; secC.Font=Enum.Font.GothamBold; secC.TextSize=9; secC.TextXAlignment=Enum.TextXAlignment.Left; secC.TextYAlignment=Enum.TextYAlignment.Bottom; secC.Parent=mainBody
        local swRow=Instance.new("Frame"); swRow.Size=UDim2.new(1,-4,0,28); swRow.BackgroundTransparency=1; swRow.Parent=mainBody
        local swLay=Instance.new("UIListLayout"); swLay.FillDirection=Enum.FillDirection.Horizontal; swLay.Padding=UDim.new(0,6); swLay.VerticalAlignment=Enum.VerticalAlignment.Center; swLay.Parent=swRow
        -- 1st = base (white, the original look), 2nd = current blue, then extras
        for _,hex in ipairs({"#FFFFFF","#4A8CFF","#AA55FF","#E24852","#3FB950","#FF5FA2","#FF9040","#FFD23F"}) do
            local sw=Instance.new("TextButton"); sw.Size=UDim2.new(0,22,0,22); sw.Text=""; sw.AutoButtonColor=false; sw.BackgroundColor3=hexToColor3(hex); sw.Parent=swRow; corner(sw,6); stroke(sw,Theme.Stroke,1,0.2)
            sw.MouseButton1Click:Connect(function() setMenuAccent(hex) end)
        end
        makeMainTextBox(mainBody,"Custom Color (HTML)",Config.AccentHex or "#4A8CFF","#RRGGBB",function(v)
            if v~="" and not setMenuAccent(v) then
                pcall(function() ShowNotification("MENU","Invalid HTML color (use #RRGGBB)") end)
            end
        end)

        -- ===== BACKGROUND IMAGE =========================================
        local secB=Instance.new("TextLabel"); secB.Size=UDim2.new(1,-4,0,22); secB.BackgroundTransparency=1; secB.Text="BACKGROUND IMAGE (all menus)"; secB.TextColor3=Theme.Dim; secB.Font=Enum.Font.GothamBold; secB.TextSize=9; secB.TextXAlignment=Enum.TextXAlignment.Left; secB.TextYAlignment=Enum.TextYAlignment.Bottom; secB.Parent=mainBody
        makeMainTextBox(mainBody,"Image Link",Config.MenuBgImage or "","URL or decal/asset id",function(v)
            Config.MenuBgImage=v; saveConfig(); applyMenuBackground(v)
        end)
        makeQuickSlider(mainBody,"Image Visibility",0,100,Config.MenuBgOpacity or 65,function(v)
            Config.MenuBgOpacity=math.floor(v); saveConfig()
            if Config.MenuBgImage and Config.MenuBgImage~="" then applyMenuBackground(Config.MenuBgImage) end
        end,"%")
        makeMainButton(mainBody,"Remove Background",function()
            Config.MenuBgImage=""; saveConfig(); applyMenuBackground("")
        end)
    end

    local idx = 0
    for _, child in ipairs(mainBody:GetChildren()) do
        if child:IsA("GuiObject") and not child:IsA("UIListLayout") and not child:IsA("UIPadding") then
            idx = idx + 1
            child.LayoutOrder = idx
        end
    end
end

-- ===== MENU ACCENT COLOR (HTML hex) + BACKGROUND IMAGE ==================
function hexToColor3(hex)
    if typeof(hex)=="Color3" then return hex end
    if type(hex)~="string" then return nil end
    hex=hex:gsub("#",""):gsub("%s","")
    if #hex==3 then hex=hex:sub(1,1):rep(2)..hex:sub(2,2):rep(2)..hex:sub(3,3):rep(2) end
    if #hex~=6 or hex:find("%X") then return nil end
    return Color3.fromRGB(tonumber(hex:sub(1,2),16),tonumber(hex:sub(3,4),16),tonumber(hex:sub(5,6),16))
end

function setMenuAccent(hex)
    local c=hexToColor3(hex); if not c then return false end
    local t=Themes.Nexus
    t.Accent=c; t.ToggleOn=c; t.SliderFill=c; t.SidebarActiveText=c
    t.SoftAccent=c:Lerp(Color3.fromRGB(19,21,26),0.78); t.SoftAccentHover=c:Lerp(Color3.fromRGB(19,21,26),0.68)
    -- keep the check mark readable on light accents (e.g. white)
    local lum=0.299*c.R+0.587*c.G+0.114*c.B
    t.ToggleKnobOn=(lum>0.7) and Color3.fromRGB(18,18,20) or Color3.fromRGB(255,255,255)
    if Config then Config.AccentHex=(type(hex)=="string" and hex or Config.AccentHex); if saveConfig then saveConfig() end end
    pcall(applyTheme,"Nexus")
    return true
end

function applyMenuBackground(link)
    local op=1-math.clamp((Config and Config.MenuBgOpacity or 65),0,100)/100
    local resolved=nil
    if link and link~="" then
        link=tostring(link)
        if link:match("^%d+$") then resolved="rbxassetid://"..link
        elseif link:match("^rbxasset") then resolved=link
        elseif link:match("^http") then
            -- executors can't load raw http images: download once, load as custom asset
            pcall(function()
                local data=game:HttpGet(link)
                writefile("mynxx_bg_img",data)
                if getcustomasset then resolved=getcustomasset("mynxx_bg_img") end
            end)
        else resolved=link end
    end
    local targets={main,actionSettingsPanel,tpSpeedSettingsPanel}
    if panels then for _,p in pairs(panels) do table.insert(targets,p) end end
    for _,f in ipairs(targets) do
        if typeof(f)=="Instance" and f:IsA("Frame") and f.Parent==gui then
            local img=f:FindFirstChild("MynxxBgImage")
            if not resolved then
                if img then img:Destroy() end
            else
                if not img then
                    img=Instance.new("ImageLabel"); img.Name="MynxxBgImage"; img.Size=UDim2.new(1,0,1,0); img.BackgroundTransparency=1; img.ScaleType=Enum.ScaleType.Crop; img.ZIndex=-5; img.BorderSizePixel=0
                    local ic=Instance.new("UICorner"); ic.CornerRadius=UDim.new(0,14); ic.Parent=img
                    img.Parent=f
                end
                img.Image=resolved; img.ImageTransparency=op
            end
            -- let the image show through the solid surfaces
            local hasImg=(resolved~=nil)
            if f==main then
                local sb=f:FindFirstChild("Sidebar"); if sb then sb.BackgroundTransparency=hasImg and 0.45 or UITransparency.Sidebar end
                local cb=f:FindFirstChild("ContentBg"); if cb then cb.BackgroundTransparency=hasImg and 0.5 or UITransparency.Content end
                f.BackgroundTransparency=hasImg and 0.35 or UITransparency.MainPanel
            else
                f.BackgroundTransparency=hasImg and 0.4 or UITransparency.Panel
            end
        end
    end
end
_G.setMenuAccent=setMenuAccent
_G.applyMenuBackground=applyMenuBackground

function _G.updateResizeGrips()
    for _, g in ipairs(_G.__MynxxGrips or {}) do
        if g and g.Parent then g.Visible = not UI.Locked end
    end
end

for tabName,btn in pairs(tabButtons) do btn.MouseButton1Click:Connect(function() loadTab(tabName) end) end

-- BOTTOM BAR
bottomBar=Instance.new("Frame"); bottomBar.Size=UDim2.new(0,0,0,42); bottomBar.AutomaticSize=Enum.AutomaticSize.X; bottomBar.AnchorPoint=Vector2.new(0.5,0); bottomBar.Position=UDim2.new(0.5,0,1,-125); bottomBar.BackgroundColor3=Theme.Background; bottomBar.BackgroundTransparency=UITransparency.BottomBar; bottomBar.BorderSizePixel=0; bottomBar.Parent=gui; corner(bottomBar,12); addOutline(bottomBar)
barPad=Instance.new("UIPadding"); barPad.PaddingLeft=UDim.new(0,12); barPad.PaddingRight=UDim.new(0,12); barPad.Parent=bottomBar
barLay=Instance.new("UIListLayout"); barLay.FillDirection=Enum.FillDirection.Horizontal; barLay.VerticalAlignment=Enum.VerticalAlignment.Center; barLay.Padding=UDim.new(0,12); barLay.SortOrder=Enum.SortOrder.LayoutOrder; barLay.Parent=bottomBar
_G.updateLogoImage = function() end
topInfo=Instance.new("Frame"); topInfo.Name="TopInfo"; topInfo.LayoutOrder=1; topInfo.Size=UDim2.new(0,0,1,0); topInfo.AutomaticSize=Enum.AutomaticSize.X; topInfo.BackgroundTransparency=1; topInfo.Parent=bottomBar
topLay=Instance.new("UIListLayout"); topLay.FillDirection=Enum.FillDirection.Horizontal; topLay.VerticalAlignment=Enum.VerticalAlignment.Center; topLay.Padding=UDim.new(0,10); topLay.SortOrder=Enum.SortOrder.LayoutOrder; topLay.Parent=topInfo

-- ÍCONE REDONDO FIXO (imagem antes dos textos)
local logoImg = Instance.new("ImageLabel")
logoImg.Name = "LogoImage"
logoImg.Size = UDim2.new(0, 28, 0, 28)
logoImg.BackgroundTransparency = 1
logoImg.Image = "rbxassetid://91373695539293"  -- ID da sua imagem
logoImg.LayoutOrder = 0
logoImg.Parent = topInfo
-- Deixa redondo
local logoCorner = Instance.new("UICorner")
logoCorner.CornerRadius = UDim.new(1, 0)
logoCorner.Parent = logoImg

lg=Instance.new("TextLabel"); lg.Name="BrandTitle"; lg.LayoutOrder=1; lg.AutomaticSize=Enum.AutomaticSize.X; lg.Size=UDim2.new(0,0,1,0); lg.BackgroundTransparency=1; lg.Text="Sxe Remasterd"; lg.TextColor3=Color3.new(1,1,1); lg.Font=Enum.Font.GothamBlack; lg.TextSize=19; lg.TextXAlignment=Enum.TextXAlignment.Left; lg.Parent=topInfo
local grad=Instance.new("UIGradient",lg); grad.Color=ColorSequence.new({ColorSequenceKeypoint.new(0,Color3.fromRGB(139,90,20)),ColorSequenceKeypoint.new(0.3,Color3.fromRGB(255,200,50)),ColorSequenceKeypoint.new(0.5,Color3.fromRGB(255,255,200)),ColorSequenceKeypoint.new(0.7,Color3.fromRGB(255,200,50)),ColorSequenceKeypoint.new(1,Color3.fromRGB(139,90,20))}); grad.Offset=Vector2.new(0,0)
task.spawn(function()while lg and lg.Parent do for o=-0.8,0.8,0.015 do if not lg or not lg.Parent then break end grad.Offset=Vector2.new(o,0);task.wait(0.02)end;grad.Offset=Vector2.new(-0.8,0)end end)
dd=Instance.new("TextLabel"); dd.Name="BrandSep"; dd.LayoutOrder=2; dd.AutomaticSize=Enum.AutomaticSize.X; dd.Size=UDim2.new(0,0,1,0); dd.BackgroundTransparency=1; dd.Text="|"; dd.TextColor3=Theme.Dim; dd.Font=Enum.Font.GothamBlack; dd.TextSize=18; dd.Parent=topInfo
dc=Instance.new("TextLabel"); dc.Name="BrandDiscord"; dc.LayoutOrder=3; dc.AutomaticSize=Enum.AutomaticSize.X; dc.Size=UDim2.new(0,0,1,0); dc.BackgroundTransparency=1; dc.Text=""; dc.TextColor3=Theme.Dim; dc.Font=Enum.Font.GothamBold; dc.TextSize=13; dc.TextXAlignment=Enum.TextXAlignment.Left; dc.Parent=topInfo
rightDiv=Instance.new("Frame"); rightDiv.Name="BottomBarDiv"; rightDiv.LayoutOrder=2; rightDiv.Size=UDim2.new(0,1,0,28); rightDiv.BackgroundColor3=Theme.Stroke; rightDiv.BackgroundTransparency=0.35; rightDiv.BorderSizePixel=0; rightDiv.Parent=bottomBar
fpsText=Instance.new("TextLabel"); fpsText.Name="FpsText"; fpsText.LayoutOrder=3; fpsText.AutomaticSize=Enum.AutomaticSize.X; fpsText.Size=UDim2.new(0,0,1,0); fpsText.BackgroundTransparency=1; fpsText.Text="FPS: --\nPING: --ms"; fpsText.TextColor3=Theme.Dim; fpsText.Font=Enum.Font.GothamBold; fpsText.TextSize=10; fpsText.TextXAlignment=Enum.TextXAlignment.Left; fpsText.Parent=bottomBar
if _G.addLazyUI then _G.addLazyUI(bottomBar, true) end

task.defer(function()
    loadTab("Auto TP")
    rebuildActions(); rebuildActionSettings(); rebuildTpSpeedSettings()
end)

-- FPS/PING COUNTER
frames,lastT=0,tick()
RunService.RenderStepped:Connect(function() frames=frames+1; local now=tick(); if now-lastT>=1 then local fps=frames; _G.currentFPS=fps; frames=0; lastT=now; local ping=0; pcall(function() ping=math.floor(LocalPlayer:GetNetworkPing() * 1000) end); fpsText.Text="FPS: "..fps.."\nPING: "..ping.."ms" end end)

-- INPUT HANDLER (NO Steal Speed, NO Unwalk keybinds)
UIS.InputBegan:Connect(function(input,gp)
    if input.UserInputType~=Enum.UserInputType.Keyboard then return end
    local kn=input.KeyCode.Name
    -- Reset (X) always works, even during steal / gameProcessed UI
    if Keybinds["Reset"] and Keybinds["Reset"]~="NONE" and kn==Keybinds["Reset"] then
        local resetFn = _G.MynxxRegenerate or _G.executeReset or executeReset or _G.InstantReset or instantReset
        if type(resetFn) == "function" then pcall(resetFn) end
        return
    end
    if gp then return end
    if input.KeyCode==UI.OpenMenuKey then
        if main.Visible then
            closeAnim(main)
            if tpSpeedSettingsPanel and tpSpeedSettingsPanel.Visible then closeAnim(tpSpeedSettingsPanel) end
            if actionSettingsPanel and actionSettingsPanel.Visible then closeAnim(actionSettingsPanel) end
            local apCmds = gui_sg and gui_sg:FindFirstChild("AdminPanelCmds")
            if apCmds and apCmds.Visible then apCmds.Visible = false end
            local clickCmds = gui_sg and gui_sg:FindFirstChild("ClickToAPCmds")
            if clickCmds and clickCmds.Visible then clickCmds.Visible = false end
            local sbOwnerCmds = gui_sg and gui_sg:FindFirstChild("SpamBaseOwnerCmds")
            if sbOwnerCmds and sbOwnerCmds.Visible then sbOwnerCmds.Visible = false end
        else openAnim(main) end
    return end
    local kn=input.KeyCode.Name
    local actions={
        ["Clone"]=instantClone,
        ["Carpet Boost"]=function() setCarpetSpeed(not CarpetState.enabled) end,
        ["Float"]=function() if _G.toggleFloat then _G.toggleFloat() end end,
        ["Reset"]=executeReset, ["Kick"]=kickPlayer,
        ["Proximity"]=function() ProximityAPActive=not ProximityAPActive; setToggle("Proximity",ProximityAPActive) end,
        ["Ragdoll Self"]=function() requestRagdollSelf() end,
        ["Invisible Steal"]=function() if _G.toggleInvisibleSteal then pcall(_G.toggleInvisibleSteal) end end,
        ["Manual TP"]=function() task.spawn(runAutoSnipe) end,
        ["Drop Brainrot"] = runDropBrainrot,
        ["Auto Buy"]=function() toggleAutoBuy() end,
        ["Click to AP"]=function()
            Config.ClickToAP = not Config.ClickToAP
            saveConfig()
            setToggle("Click to AP", Config.ClickToAP)
            setToggle("ClickToAP", Config.ClickToAP)
            ShowNotification("CLICK TO AP", Config.ClickToAP and "Enabled" or "Disabled")
        end,
        ["WalkSpeed"]=function() setWalkSpeedEnabled(not WalkSpeedState.enabled) end,
    }
    for actionName,keyBind in pairs(Keybinds) do if keyBind==kn and actions[actionName] then actions[actionName](); return end end
end)


if Config.TpSettings.TpOnLoad then
    _G.TpOnLoadRunning = true
    task.spawn(function()
        local char = LocalPlayer.Character
        if not char then
            char = LocalPlayer.CharacterAdded:Wait(5)
        end
        local hrp = char and char:WaitForChild("HumanoidRootPart", 5)
        local hum = char and char:WaitForChild("Humanoid", 5)
        if not hrp or not hum or hum.Health <= 0 then
            _G.Mynxx_Ready = true
            local ok, err = pcall(InicializarRecursos)
            if not ok then warn("Erro ao inicializar recursos: ", err) end
            return
        end
        local lastPos = hrp.Position
        local stableFrames = 0
        while stableFrames < 0.1 do
            if (hrp.Position - lastPos).Magnitude < 0.1 then
                stableFrames = stableFrames + 1
            else
                stableFrames = 0
            end
            lastPos = hrp.Position
            RunService.RenderStepped:Wait()
        end
        local userDelay = Config.TpSettings.TpOnLoadDelay or 3
        userDelay = math.min(userDelay, 30)
        if userDelay > 0 then
            local freezeConn = RunService.Heartbeat:Connect(function()
                if hrp and hrp.Parent then
                    hrp.AssemblyLinearVelocity = Vector3.zero
                    hrp.AssemblyAngularVelocity = Vector3.zero
                    hrp.CFrame = hrp.CFrame
                end
            end)
            task.wait(userDelay)
            freezeConn:Disconnect()
        end
        local scannerReady = false
        local t0 = os.clock()
        if Config.TpSettings.GrabbleTP then
            while not _G.MynxxScannerReady and (os.clock() - t0) < 5 do
                task.wait(0.05)
            end
            scannerReady = _G.MynxxScannerReady or false
        else
            while (not SharedState.InitialScanComplete or #SharedState.AllAnimalsCache == 0) and (os.clock() - t0) < 5 do
                task.wait(0.05)
            end
            scannerReady = SharedState.InitialScanComplete and #SharedState.AllAnimalsCache > 0
        end
        if not scannerReady then
            warn("[TP On Load] Scanner não ficou pronto a tempo, tentando TP mesmo assim.")
        end
        local function doTP()
            if _G.MynxxStartSideTP then
                pcall(_G.MynxxStartSideTP)
            else
                pcall(runAutoSnipe)
            end
        end
        local success, err = pcall(doTP)
        if not success then
            warn("[TP On Load] Erro no TP: ", err)
            task.wait(0.5)
            pcall(doTP)
        end
        _G.Mynxx_Ready = true
        local ok, errInit = pcall(InicializarRecursos)
        if not ok then
            warn("Erro ao inicializar recursos: ", errInit)
        end
    end)
else
    _G.Mynxx_Ready = true
    local ok, err = pcall(InicializarRecursos)
    if not ok then
        warn("Falha ao chamar InicializarRecursos: ", err)
        task.delay(0.5, function()
            pcall(InicializarRecursos)
        end)
    end
end

_G.InvisStealAngle=Config.InvisStealAngle or 180; _G.SinkSliderValue=Config.SinkSliderValue or 8
_G.AutoRecoverLagback=true; _G.AutoInvisDuringSteal=Config.AutoInvisDuringSteal or false

print("Sxe Remasterd Loaded")


task.spawn(function()
    while true do
        task.wait(15)
        pcall(collectgarbage, "collect")
    end
end)

RunService.RenderStepped:Connect(function()
    local isRag, ragLeft = checkRagdollBar()
    if isRag then
        hudStroke.Color = RAG_RED
        hudGlow.Color = RAG_RED
        hudProgressFill.BackgroundColor3 = RAG_RED
        if ragLeft and ragLeft > 0 then
            local endTime = LocalPlayer:GetAttribute("RagdollEndTime")
            if endTime ~= ragLastEndTime then
                ragLastEndTime = endTime
                ragTotalDuration = ragLeft
            end
            local p = math.clamp(ragLeft / math.max(ragTotalDuration or ragLeft, 0.1), 0, 1)
            hudName.Text = string.format("RAGDOLL - %.1fs", ragLeft)
            hudProgressFill.Size = UDim2.new(p, 0, 1, 0)
            hudPercent.Text = string.format("%.1fs", ragLeft)
            hudPercent.TextColor3 = RAG_ORANGE
        else
            hudName.Text = "RAGDOLL"
            hudProgressFill.Size = UDim2.new(1, 0, 1, 0)
            hudPercent.Text = "..."
            hudPercent.TextColor3 = RAG_ORANGE
        end
        return
    else
        hudStroke.Color = STEALBAR.STROKE
        hudGlow.Color = STEALBAR.GLOW
        hudPercent.TextColor3 = STEALBAR.TEXT
        hudName.TextColor3 = STEALBAR.TEXT -- RESETA A COR DO TEXTO PARA BRANCO (PADRÃO)
    end

    if not (Config and Config.AutoStealEnabled) then
        hudName.Text = "Disabled"
        hudProgressFill.Size = UDim2.new(0, 0, 1, 0)
        hudPercent.Text = "0%"
        hudProgressFill.BackgroundColor3 = STEALBAR.FILL1
        return
    end

    if LocalPlayer:GetAttribute("Stealing") then
        hudProgressFill.Size = UDim2.new(1, 0, 1, 0)
        hudProgressFill.BackgroundColor3 = Color3.fromRGB(100, 220, 130)
        hudPercent.Text = "100%"
        hudName.Text = "Carrying Brainrot!"
        hudName.TextColor3 = STEALBAR.TEXT -- TEXTO BRANCO
        return
    end

    local status = _G.Mynxx_StealStatus or {}
    if status.active then
        local p = math.clamp((tick() - (status.start or 0)) / (status.duration or 1.3), 0, 1)
        hudProgressFill.Size = UDim2.new(p, 0, 1, 0)
        hudPercent.Text = math.floor(p * 100) .. "%"
        hudProgressFill.BackgroundColor3 = (p >= 1) and Color3.fromRGB(100, 220, 130) or STEALBAR.FILL1
        hudName.Text = "Stealing..."
        hudName.TextColor3 = STEALBAR.TEXT -- TEXTO BRANCO
    elseif status.target then
        hudProgressFill.Size = UDim2.new(0, 0, 1, 0)
        hudPercent.Text = "0%"
        hudProgressFill.BackgroundColor3 = STEALBAR.FILL1
        local vs = _pbFmtVal(status.target.mps or status.target.value)
        hudName.Text = (status.target.name or "Brainrot") .. ((vs ~= "0") and (" - $" .. vs) or "")
        hudName.TextColor3 = STEALBAR.TEXT -- TEXTO BRANCO
    else
        hudProgressFill.Size = UDim2.new(0, 0, 1, 0)
        hudPercent.Text = "0%"
        hudProgressFill.BackgroundColor3 = Color3.fromRGB(0, 0, 0) -- ALTERADO: Barra de preenchimento PRETA (para combinar com a imagem)
        hudName.Text = "No items"
        hudName.TextColor3 = Color3.fromRGB(255, 50, 50) -- ALTERADO: Texto "No items" em VERMELHO (como na imagem)
    end
end)

function doTP()
    local hrp = LocalPlayer.Character and LocalPlayer.Character:FindFirstChild("HumanoidRootPart")
    if not hrp then return end
    pcall(function()
        if sethiddenproperty then
            local cf = hrp.CFrame
            sethiddenproperty(hrp, "CFrame", CFrame.new(cf.Position.X + 500, cf.Position.Y, cf.Position.Z + 500) * (cf - cf.Position))
            task.wait(0.23)
            sethiddenproperty(hrp, "CFrame", cf)
        end
    end)
end

-- No Player Collision Logic
lastNoclipUpdate = 0
RunService.Stepped:Connect(function()
    local now = tick()
    if now - lastNoclipUpdate < 0.1 then return end
    lastNoclipUpdate = now
    for _, p in ipairs(Players:GetPlayers()) do
        if p ~= LocalPlayer and p.Character then
            for _, part in ipairs(p.Character:GetDescendants()) do
                if part:IsA("BasePart") then 
                    part.CanCollide = false 
                end
            end
        end
    end
end)
Mynxx_AC = {} 
end 
   end

-- ============================================================
-- FUNÇÃO QUE INICIALIZA TODOS OS RECURSOS PESADOS (antigo task.spawn)
-- ============================================================
function InicializarRecursos()
    -- ============================================================
    -- 1) Coisas leves (tema, UI) – essas podem continuar sequenciais
    -- ============================================================
    pcall(function()
        if Config and Config.AccentHex and setMenuAccent then
            setMenuAccent(Config.AccentHex)
        else
            applyTheme("Nexus")
        end
    end)
    pcall(function()
        if Config and Config.MenuBgImage and Config.MenuBgImage ~= "" and applyMenuBackground then
            applyMenuBackground(Config.MenuBgImage)
        end
    end)
    pcall(function() if initToggles then initToggles() end end)
    pcall(function() if _G.updateResizeGrips then _G.updateResizeGrips() end end)

    -- ============================================================
    -- 2) Ativações médias (Infinite Jump, Dark Theme, etc.) – também em paralelo
    -- ============================================================
    task.spawn(function()
        if Config.InfiniteJump then setInfiniteJump(true) end
        if Config.DarkTheme and setDarkTheme then setDarkTheme(true) end
        if Config.Unwalk then setUnwalk(true) end
        Workspace.CurrentCamera.FieldOfView = Config.FOV or 70
    end)

    -- ============================================================
    -- 3) Recursos pesados (FPS Boost, X-Ray, Anti-Ragdoll, Steal Mode) – TODOS EM PARALELO
    -- ============================================================
    task.spawn(function()
        if Config.FPSBoostUltra then setFPSBoostUltra(true) end
    end)

    task.spawn(function()
        if Config.KashiOptimizer then setKashiOptimizer(true) end
    end)

    task.spawn(function()
        if Config.XRay then setXRay(true) end
    end)

    task.spawn(function()
        if Config.AntiRagdoll then startAntiRagdoll() end
    end)

    task.spawn(function()
        if Config.StealHighest then setStealMode("Highest")
        elseif Config.StealPriority then setStealMode("Priority")
        elseif Config.StealNearest then setStealMode("Nearest")
        else setStealMode("Highest") end
    end)

    -- Opcional: marcar como pronto
    _G.Mynxx_Ready = true
    print("Sxe Remasterd Loaded")
end

-- ============================================================
-- Mynxx AUTO-STEAL + CLONE-TP ENGINE (integrated) -- xentp.lua 1:1 port
-- Speed: Config.TpSettings.GrabbleTPSpeed | Clone Delay: Config.TpSettings.CloneDelayVal
-- ============================================================
do
    local Players    = game:GetService("Players")
    local RunService = game:GetService("RunService")
    local UIS        = game:GetService("UserInputService")
    local RS         = game:GetService("ReplicatedStorage")
    local LP = Players.LocalPlayer

    -- ===== Modules =====
    local Synchronizer, AnimalsData, AnimalsShared, NumberUtils
    local function loadModules()
        if Synchronizer then return true end
        local ok = pcall(function()
            local Packages = RS:WaitForChild("Packages", 5)
            local Datas = RS:WaitForChild("Datas", 5)
            local Shared = RS:WaitForChild("Shared", 5)
            local Utils = RS:WaitForChild("Utils", 5)
            Synchronizer = require(Packages:WaitForChild("Synchronizer"))
            AnimalsData = require(Datas:WaitForChild("Animals"))
            AnimalsShared = _G._xenAnimShim
            NumberUtils = require(Utils:WaitForChild("NumberUtils"))
        end)
        return ok and Synchronizer ~= nil
    end

    local NetModule
    local function loadNet()
        if NetModule then return true end
        local ok, mod = pcall(function()
            return _G.Net
        end)
        if not ok or type(mod) ~= "table" then return false end
        NetModule = mod
        return true
    end

    local function fireGrapple()
        if not NetModule then loadNet() end
        if not NetModule then return end
        local char = LP.Character
        if not char then return end
        if not char:FindFirstChild("Grapple Hook") then
            local bp = LP:FindFirstChild("Backpack")
            local tool = bp and bp:FindFirstChild("Grapple Hook")
            local hum = char:FindFirstChildOfClass("Humanoid")
            if tool and hum then pcall(function() hum:EquipTool(tool) end) end
        end
        if not char:FindFirstChild("Grapple Hook") then return end
        pcall(function() NetModule:RemoteEvent("UseItem"):FireServer(2) end)
    end
    _G.MynxxFireGrapple = fireGrapple

    -- ===== Speed config (live-settable) =====
    local MynxxSpeed = { CARPET = 450, INBASE = 350 }
    _G.MynxxSetCarpetSpeed = function(v) v = tonumber(v); if v and v > 0 then MynxxSpeed.CARPET = v end end
    _G.MynxxSetInbaseSpeed = function(v) v = tonumber(v); if v and v > 0 then MynxxSpeed.INBASE = v end end
    _G.MynxxGetCarpetSpeed = function() return MynxxSpeed.CARPET end
    if Config and Config.TpSettings then
        if tonumber(Config.TpSettings.GrabbleTPSpeed) then MynxxSpeed.CARPET = tonumber(Config.TpSettings.GrabbleTPSpeed) end
    end

    -- ===== Tools =====
    local CARPET_NAMES = { "Flying Carpet", "Carpet", "Cloud", "Witch's Broom", "Cupid's Wings", "Santa's Sleigh", "Magic Carpet" }
    local GRAPPLE_NAMES = { "Grapple Hook", "Grappling Hook", "Grapple", "Hook", "Web Slinger", "Grapple Gun", "GrappleHook" }
    local function findTool(name)
        local char = LP.Character
        local bp = LP:FindFirstChild("Backpack")
        return (char and char:FindFirstChild(name)) or (bp and bp:FindFirstChild(name))
    end
    local function findGrapple()
        for _, n in ipairs(GRAPPLE_NAMES) do
            local t = findTool(n); if t then return t end
        end
        return nil
    end
    local function equipCarpet()
        local char = LP.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not hum then return nil end
        local preferred = Config and Config.TpSettings and Config.TpSettings.Tool
        if preferred then
            local pt = findTool(preferred)
            if pt and pt:IsA("Tool") then
                if pt.Parent ~= char then pcall(function() hum:EquipTool(pt) end) end
                return preferred
            end
        end
        for _, n in ipairs(CARPET_NAMES) do
            local t = findTool(n)
            if t and t:IsA("Tool") then
                if t.Parent ~= char then pcall(function() hum:EquipTool(t) end) end
                return n
            end
        end
        return nil
    end
    local function carpetEngage()
        if not NetModule then pcall(loadNet) end
        local _t0 = os.clock()
        while not findTool("Grapple Hook") and os.clock() - _t0 < 5 do
            if not NetModule then pcall(loadNet) end
            RunService.Heartbeat:Wait()
        end
        local char = LP.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not char or not hum then return nil end
        if not char:FindFirstChild("Grapple Hook") then
            local g = findTool("Grapple Hook")
            if g then pcall(function() hum:EquipTool(g) end) end
        end
        task.wait(0.08)
        if NetModule and LP.Character and LP.Character:FindFirstChild("Grapple Hook") then
            pcall(function() NetModule:RemoteEvent("UseItem"):FireServer(2) end)
        end
        task.wait(0.15)
        local h = LP.Character and LP.Character:FindFirstChildOfClass("Humanoid")
        if h then pcall(function() h:UnequipTools() end) end
        task.wait(0.15)
        local cn
        local _tc = os.clock()
        repeat
            cn = equipCarpet()
            local c = LP.Character
            if cn and c and c:FindFirstChild(cn) then break end
            RunService.Heartbeat:Wait()
        until os.clock() - _tc > 1
        return cn
    end

    -- ===== Priority system =====
    local PET_PRIORITY_TIERS = {
        [1]  = { pets = {"Headless Horseman"}, threshold = 0 },
        [2]  = { pets = {"Signore Carapace"}, threshold = 0 },
        [3]  = { pets = {"Strawberry Elephant"}, threshold = 0 },
        [4]  = { pets = {"Arcadragon"}, threshold = 0 },
        [5]  = { pets = {"Elefanto Frigo"}, threshold = 5e9 },
        [6]  = { pets = {"John Pork"}, threshold = 10e9 },
        [7]  = { pets = {"Meowl"}, threshold = 5e9 },
        [8]  = { pets = {"Skibidi Toilet"}, threshold = 5e9 },
        [9]  = { pets = {"Love Love Bear"}, threshold = 0 },
        [10] = { pets = {"Antonio"}, threshold = 0 },
        [11] = { pets = {"Pancake and Syrup"}, threshold = 0 },
        [12] = { pets = {"Griffin"}, threshold = 0 },
        [13] = { pets = {"La Supreme Combinasion","Fishino Clownino","Dragon Gingerini","Tirilikalika Tirilikalako"}, threshold = 5e9 },
        [14] = { pets = {"Ginger Gerat","Pet"}, threshold = 10e9 },
        [15] = { pets = {"Hydra Bunny","Digi Narwhal","Kalika Bros"}, threshold = 3e9 },
        [16] = { pets = {"Hydra Dragon Cannelloni","Dragon Cannelloni","Bunny and Eggy"}, threshold = 3e9 },
        [17] = { pets = {"Globa Steppa","Ketupat Bros","Rosey and Teddy","La Casa Boo","Fragola la la"}, threshold = 3e9 },
        [18] = { pets = {"Fragola La La La","Cerberus","Guest 666","Los Hackers"}, threshold = 1e9 },
        [19] = { pets = {"Garama and Madundung","Spooky and Pumpky","Reinito Sleighito","Burguro And Fryuro","Cooki and Milki","Fragrama and Chocrama","La Food Combinasion","Los Amigos","Foxini Lanternini","Capitano Moby","Fortunu and Cashuru","Los Sekolahs","Celestial Pegasus"}, threshold = 750e6 },
        [20] = { pets = {"La Secret Combinasion","Sammyni Fattini","Cloverat Clapat","Popcuru and Fizzuru"}, threshold = 1e9 },
    }
    local TIER_LOOKUP = {}
    for tier, data in pairs(PET_PRIORITY_TIERS) do
        for _, name in ipairs(data.pets) do TIER_LOOKUP[name] = tier end
    end
    local LOCKED_TIERS = { [1]=true,[2]=true,[3]=true,[4]=true }
    local DIRECT_THRESHOLDS = {
        [3] = { [4] = 10e9 },
        [4] = {},
        [5] = { [6] = math.huge },
        [6] = { [9] = math.huge, [10] = math.huge, [12] = 15e9 },
        [10] = { [12] = 20e9 },
        [11] = { [12] = 10e9 },
    }
    local MUTATION_PRIORITY = {
        ["Galaxy"]=1,["Candy"]=1,["Yin Yang"]=1,["YinYang"]=1,["Divine"]=1,
        ["Cursed"]=1,["Lava"]=1,["Radioactive"]=1,["Cyber"]=1,["Rainbow"]=1,["Bloodrot"]=2,
    }
    local MUTATED_BEATS_GRIFFIN = {
        ["Fishino Clownino"]=true,["Globa Steppa"]=true,
        ["La Supreme Combinasion"]=true,["Tirilikalika Tirilikalako"]=true,
    }
    local function getMutPrio(m)
        if not m or m == "" or m == "None" then return 0 end
        if MUTATION_PRIORITY[m] then return MUTATION_PRIORITY[m] end
        local n = tostring(m):lower():gsub("[%s%-_]","")
        if n == "bloodrot" then return 2 end
        if n == "yinyang" or n == "galaxy" or n == "candy" or n == "divine"
            or n == "cursed" or n == "lava" or n == "radioactive" or n == "cyber"
            or n == "rainbow" then return 1 end
        return 0
    end
    local function getCumThreshold(hi, lo)
        if DIRECT_THRESHOLDS[hi] and DIRECT_THRESHOLDS[hi][lo] then return DIRECT_THRESHOLDS[hi][lo] end
        if LOCKED_TIERS[hi] then return math.huge end
        local total = 0
        for t = hi + 1, lo do
            local td = PET_PRIORITY_TIERS[t]
            if td and td.threshold > 0 then total = total + td.threshold end
        end
        return total
    end
    local function petOutranks(aName, bName, aMut, bMut, aMPS, bMPS)
        if aName == "Strawberry Elephant" and bName == "John Pork" then return true end
        if aName == "John Pork" and bName == "Strawberry Elephant" then return false end
        if MUTATED_BEATS_GRIFFIN[aName] and bName == "Griffin" and getMutPrio(aMut) >= 1 then return true end
        if aName == "Griffin" and MUTATED_BEATS_GRIFFIN[bName] and getMutPrio(bMut) >= 1 then return false end
        if aName == "Antonio" and bName == "Elefanto Frigo" and getMutPrio(aMut) >= 1 then return true end
        if aName == "Elefanto Frigo" and bName == "Antonio" and getMutPrio(bMut) >= 1 then return false end
        local tA = TIER_LOOKUP[aName] or 99
        local tB = TIER_LOOKUP[bName] or 99
        if not (TIER_LOOKUP[aName] and TIER_LOOKUP[bName]) then
            if tA == tB then return (aMPS or 0) > (bMPS or 0) end
            return tA < tB
        end
        if tA == tB then
            local pA, pB = getMutPrio(aMut), getMutPrio(bMut)
            if pA ~= pB then return pA > pB end
            return (aMPS or 0) > (bMPS or 0)
        end
        if tA == 4 and tB == 3 then return true end
        if tA == 3 and tB == 4 then return false end
        local hi = math.min(tA, tB)
        local lo = math.max(tA, tB)
        local hiMPS = tA < tB and aMPS or bMPS
        local loMPS = tA < tB and bMPS or aMPS
        local cum = getCumThreshold(hi, lo)
        if cum > 0 and cum ~= math.huge then
            if (loMPS or 0) - (hiMPS or 0) > cum then return tA > tB end
        end
        return tA < tB
    end

    -- ===== Plot/Channel helpers =====
    local function getPlotChannel(plotName)
        if not Synchronizer then return nil end
        local channel
        pcall(function() channel = _G.XenSyncGet(plotName) end)
        -- timeout so an unsynced plot can't stall a whole scan for seconds at boot
        if not channel then local _t0=os.clock() repeat channel=_G.XenSyncGet(plotName) if channel then break end task.wait(0.05) until os.clock()-_t0>1 end
        return channel
    end
    local function channelGet(channel, key)
        if not channel then return nil end
        local v
        pcall(function() if type(channel.Get) == "function" then v = channel:Get(key) end end)
        if v == nil then pcall(function() v = channel.CacheTable and channel.CacheTable[key] end) end
        return v
    end
    local function isMyPlot(channel)
        if not channel then return false end
        local owner = channelGet(channel, "Owner")
        if not owner then return false end
        local result = false
        pcall(function()
            if typeof(owner) == "Instance" and owner:IsA("Player") then
                result = owner.UserId == LP.UserId
            elseif type(owner) == "table" and owner.UserId then
                result = owner.UserId == LP.UserId
            elseif typeof(owner) == "Instance" then
                result = owner == LP
            end
        end)
        return result
    end
    local function ownerInGame(channel)
        if not channel then return false end
        local owner = channelGet(channel, "Owner")
        if not owner then return false end
        local inGame = false
        pcall(function()
            if typeof(owner) == "Instance" and owner:IsA("Player") then
                inGame = Players:FindFirstChild(owner.Name) ~= nil
            elseif type(owner) == "number" then
                inGame = Players:GetPlayerByUserId(owner) ~= nil
            elseif type(owner) == "table" and owner.Name then
                inGame = Players:FindFirstChild(tostring(owner.Name)) ~= nil
            elseif typeof(owner) == "Instance" and owner.Name then
                inGame = Players:FindFirstChild(owner.Name) ~= nil
            end
        end)
        return inGame
    end
    local function isPlotUnlocked(plotName)
        local ok, res = pcall(function()
            local channel = getPlotChannel(plotName)
            if not channel then return false end
            return channelGet(channel, "BlockEndTimeFirstFloor") == nil
        end)
        return ok and (res == true)
    end

    -- ===== Pet position =====
    -- Position cache: the nested GetDescendants + GetBoundingBox is the single
    -- most expensive part of scanAllPets (which runs up to 10x/s). Podium pets
    -- don't move, so cache each slot's position for 5s.
    local _petPosCache = {}
    local function getPetPosition(plot, slot)
        local key = plot.Name .. "|" .. tostring(slot)
        local hit = _petPosCache[key]
        local now = os.clock()
        if hit and (now - hit.t) < 5 then return hit.pos end
        local function compute()
            local podiums = plot:FindFirstChild("AnimalPodiums")
            if not podiums then return nil end
            local podium = podiums:FindFirstChild(tostring(slot))
            if not podium then return nil end
            for _, desc in ipairs(podium:GetDescendants()) do
                if desc:IsA("Model") and desc.Name ~= "Claim" and desc.Name ~= "Base" and desc.Name ~= "Decorations" then
                    local hasMesh = false
                    for _, c in ipairs(desc:GetDescendants()) do
                        if c:IsA("MeshPart") then hasMesh = true; break end
                    end
                    if hasMesh then
                        local ok, cf = pcall(function() return desc:GetBoundingBox() end)
                        if ok then return cf.Position end
                    end
                end
            end
            local ok, cf = pcall(function() return podium:GetPivot() end)
            if ok then return cf.Position end
            return podium.Position
        end
        local pos = compute()
        if pos then _petPosCache[key] = { pos = pos, t = now } end
        return pos
    end

    -- ===== Fusing check =====
    local _BLOCKING_MACHINE_TYPES = { Fuse=true, Duel=true, Trade=true, Crafting=true }
    local function _MynxxIsFusing(animalData)
        if type(animalData) ~= "table" then return false end
        local m = animalData.Machine
        if type(m) ~= "table" then return false end
        return _BLOCKING_MACHINE_TYPES[m.Type] == true and m.Active == true
    end

    -- ===== Scanner =====
    local function scanAllPets()
        local pets = {}
        -- `misses` counts entries the scan could NOT resolve yet (unsynced channel,
        -- unloaded module data, unrendered podium). misses == 0 => scan is COMPLETE.
        local misses = 0
        if not loadModules() then return pets, 1 end
        local Plots = workspace:FindFirstChild("Plots")
        if not Plots then return pets, 1 end
        for _, plot in ipairs(Plots:GetChildren()) do
            local channel = getPlotChannel(plot.Name)
            if not channel then misses = misses + 1 continue end
            if isMyPlot(channel) then continue end
            if not ownerInGame(channel) then continue end
            local animalList = channelGet(channel, "AnimalList")
            if not animalList then continue end
            for slot, animalData in pairs(animalList) do
                if type(animalData) ~= "table" then continue end
                local animalName = animalData.Index
                if not animalName then continue end
                local animalInfo = AnimalsData and AnimalsData[animalName]
                if not animalInfo then misses = misses + 1 continue end
                if _MynxxIsFusing(animalData) then continue end
                local mutation = animalData.Mutation or "None"
                local genValue = 0
                pcall(function()
                    genValue = AnimalsShared:GetGeneration(animalName, animalData.Mutation, animalData.Traits, nil)
                end)
                local displayName = (animalInfo and animalInfo.DisplayName) or animalName
                local pos = getPetPosition(plot, slot)
                if pos then
                    table.insert(pets, {
                        name = displayName, index = animalName, mps = genValue,
                        mutation = mutation, position = pos, plot = plot.Name, slot = tostring(slot),
                    })
                else
                    -- animal exists in the data channel but its podium isn't rendered yet
                    misses = misses + 1
                end
            end
        end
        local conveyorFolder = workspace:FindFirstChild("RenderedMovingAnimals")
        if conveyorFolder then
            for _, model in ipairs(conveyorFolder:GetChildren()) do
                pcall(function()
                    if not model:IsA("Model") then return end
                    local animalInfo = AnimalsData and AnimalsData[model.Name]
                    if not animalInfo then return end
                    local mutation = model:GetAttribute("Mutation") or "None"
                    local genValue = 0
                    pcall(function()
                        genValue = AnimalsShared:GetGeneration(model.Name, model:GetAttribute("Mutation"), nil, nil) or 0
                    end)
                    if genValue <= 0 then return end
                    -- parts may not be streamed in yet when the belt is far away:
                    -- fall back to the model pivot instead of dropping the pet
                    -- (a dropped PRIORITY pet made the TP pick a base pet instead)
                    local part = model.PrimaryPart or model:FindFirstChildWhichIsA("BasePart")
                    local pos = part and part.Position or model:GetPivot().Position
                    table.insert(pets, {
                        name = animalInfo.DisplayName or model.Name, index = model.Name,
                        mps = genValue, mutation = mutation, position = pos,
                        plot = nil, slot = nil, conveyor = true, model = model,
                    })
                end)
            end
        end
        table.sort(pets, function(a, b)
            return petOutranks(a.name, b.name, a.mutation, b.mutation, a.mps, b.mps)
        end)
        return pets, misses
    end

    -- ===== Sky coordinates =====
    local UPPER = {
        B = {{coord=Vector3.new(-487.921448,16.850713,-75.768013),facing="NORTH"},{coord=Vector3.new(-332.379730,16.850722,-75.762100),facing="NORTH"},{coord=Vector3.new(-487.134918,16.850713,-18.094154),facing="SOUTH"},{coord=Vector3.new(-316.300171,16.850713,-17.845898),facing="SOUTH"}},
        C = {{coord=Vector3.new(-330.765381,16.850713,31.424425),facing="NORTH"},{coord=Vector3.new(-502.989349,16.850713,31.172430),facing="NORTH"},{coord=Vector3.new(-489.077087,16.850713,89.010147),facing="SOUTH"},{coord=Vector3.new(-330.908936,16.850713,88.930145),facing="SOUTH"}},
        D = {{coord=Vector3.new(-331.264893,16.850713,138.209167),facing="NORTH"},{coord=Vector3.new(-487.935181,16.850713,138.026321),facing="NORTH"},{coord=Vector3.new(-487.774933,16.850713,195.882538),facing="SOUTH"},{coord=Vector3.new(-330.799133,16.850575,196.022354),facing="SOUTH"}},
    }
    local LOWER = {
        B = {{coord=Vector3.new(-335.725586,-3.048217,-74.984589),facing="NORTH"},{coord=Vector3.new(-503.214233,-3.048217,-75.043137),facing="NORTH"},{coord=Vector3.new(-483.619385,-3.718430,-18.844337),facing="SOUTH"},{coord=Vector3.new(-316.147095,-3.048218,-18.818844),facing="SOUTH"}},
        C = {{coord=Vector3.new(-335.985413,-3.048218,32.051426),facing="NORTH"},{coord=Vector3.new(-503.277008,-3.048217,31.956175),facing="NORTH"},{coord=Vector3.new(-483.749390,-3.048218,88.147003),facing="SOUTH"},{coord=Vector3.new(-315.793823,-3.048217,88.163979),facing="SOUTH"}},
        D = {{coord=Vector3.new(-335.476654,-3.048218,139.001083),facing="NORTH"},{coord=Vector3.new(-503.710083,-3.048218,138.989883),facing="NORTH"},{coord=Vector3.new(-315.654938,-3.048218,195.302444),facing="SOUTH"},{coord=Vector3.new(-483.859253,-3.048218,195.269043),facing="SOUTH"}},
    }
    local UPPER_Y_THRESHOLD = 7
    local TALL_PETS = { ["La Secret Combinasion"]=true, ["La Jolly Grande"]=true }
    local TALL_OFFSET = 3

    local function getPetFloor(petY)
        if petY > 23.15 then return 3 end
        if petY >= 11 then return 2 end
        return 1
    end

    -- Clone TP one floor below the target (étage 3 -> ciel étage 2, étage 2 -> ciel étage 1, étage 1 -> ciel sol)
    local function getCloneCoordTable(petY)
        if getPetFloor(petY) >= 3 then return UPPER end
        return LOWER
    end

    local function getGoToBrainrotTargetY(petY)
        local floor = getPetFloor(petY)
        if floor == 3 then return 21 end
        if floor == 2 then return 14.5 end
        if petY < 3 then return 0 end
        return 5.1
    end
    local BASES_LOW = {
        [1]=Vector3.new(-476.52,-2,220.94),[2]=Vector3.new(-476.52,-2,113.77),
        [3]=Vector3.new(-476.52,-2,6.18),[4]=Vector3.new(-476.52,-2,-101.07),
        [5]=Vector3.new(-342.66,-2,221.45),[6]=Vector3.new(-342.66,-2,113.41),
        [7]=Vector3.new(-342.66,-2,6.25),[8]=Vector3.new(-342.66,-2,-99.73),
    }
    local BASES_HIGH = {
        [1]=Vector3.new(-479.51,18,220.94),[2]=Vector3.new(-479.51,18,113.77),
        [3]=Vector3.new(-479.51,18,6.18),[4]=Vector3.new(-479.51,18,-101.07),
        [5]=Vector3.new(-339.48,18,221.45),[6]=Vector3.new(-339.48,18,113.41),
        [7]=Vector3.new(-339.48,18,6.25),[8]=Vector3.new(-339.48,18,-99.73),
    }
    local FRONT_Y_LOW = -3.048217
    local FRONT_Y_HIGH = 16.850713
    local COLUMN_SPLIT_X = -410
    local FRONT_Z_CLAMP = 18
    local SIDE_NEAR_Z = 45

    local function getClosestBaseIdx(pos)
        local closest, dist = 1, math.huge
        for i = 1, 8 do
            local b = BASES_LOW[i]
            local d = (pos.X - b.X)^2 + (pos.Z - b.Z)^2
            if d < dist then dist = d; closest = i end
        end
        return closest
    end
    local function buildFrontCandidate(idx, isUpper, playerZ)
        local base = isUpper and BASES_HIGH[idx] or BASES_LOW[idx]
        local frontY = isUpper and FRONT_Y_HIGH or FRONT_Y_LOW
        local frontZ = math.clamp(playerZ - base.Z, -FRONT_Z_CLAMP, FRONT_Z_CLAMP) + base.Z
        local coord = Vector3.new(base.X, frontY, frontZ)
        local faceDir = (idx <= 4) and Vector3.new(-1, 0, 0) or Vector3.new(1, 0, 0)
        return coord, faceDir
    end
    -- Reset (X): reposiciona o personagem na frente da própria base.
    -- O índice é obtido pela posição do plot pertencente ao jogador, então não
    -- depende de nomes fixos como Plot_B1/Plot_C2.
    local function resetToOwnBase()
        local char = Players.LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then return false end

        local plots = Workspace:FindFirstChild("Plots")
        if not plots then return false end

        local ownPlotPosition
        for _, plot in ipairs(plots:GetChildren()) do
            local owned = false
            pcall(function()
                owned = isMyPlot_Instant(plot.Name) == true
            end)
            if owned then
                ownPlotPosition = plot:GetPivot().Position
                break
            end
        end
        if not ownPlotPosition then return false end

        local idx = getClosestBaseIdx(ownPlotPosition)
        local coord, faceDir = buildFrontCandidate(idx, false, ownPlotPosition.Z)
        local target = coord + faceDir * 4

        pcall(function()
            hrp.Anchored = false
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
            hrp.CFrame = CFrame.lookAt(target, target - faceDir)
        end)
        return true
    end
    _G.MynxxResetToBase = resetToOwnBase

    local function plotSides(coordTable, idx)
        local base = BASES_LOW[idx]
        local isWest = idx <= 4
        local out = {}
        for _, coords in pairs(coordTable) do
            for _, data in ipairs(coords) do
                if ((data.coord.X < COLUMN_SPLIT_X) == isWest)
                   and math.abs(data.coord.Z - base.Z) < SIDE_NEAR_Z then
                    out[#out + 1] = data
                end
            end
        end
        return out
    end
    local function findClosest(petPos, coordTable)
        local best, bestKey, bestDist = nil, nil, math.huge
        for skyKey, coords in pairs(coordTable) do
            for _, data in ipairs(coords) do
                local c = data.coord
                local d = math.sqrt((petPos.X - c.X)^2 + (petPos.Z - c.Z)^2)
                if d < bestDist then bestDist = d; best = data; bestKey = skyKey end
            end
        end
        return best, bestKey
    end

    -- ===== Viz =====
    local _vizParts = {}
    local _vizBallAnimConn = nil
    local _vizGen = 0
    local _vizStickyUntil = 0
    local function clearViz(force)
        -- sticky window (goToBrainrot keeps its trace 5s): defer the clear
        if not force and os.clock() < _vizStickyUntil then
            task.delay(_vizStickyUntil - os.clock() + 0.05, function() clearViz(true) end)
            return
        end
        _vizGen = _vizGen + 1
        if _vizBallAnimConn then
            pcall(function() _vizBallAnimConn:Disconnect() end)
            _vizBallAnimConn = nil
        end
        local batch = {}
        for i, p in ipairs(_vizParts) do batch[i] = p end
        table.clear(_vizParts)
        -- straight Destroy (no fade-out tween per part): the fade spun up one
        -- TweenService tween per segment on every wipe.
        for _, p in ipairs(batch) do
            if p and p.Parent then pcall(function() p:Destroy() end) end
        end
    end
    local function _vizFadeIn(p, target)
        -- instant set (no tween): the fade-in was pure TweenService overhead.
        p.Transparency = target or 0
    end
    local function vizLine(a, b, color)
        local d = b - a
        if d.Magnitude < 0.05 then return end
        local p = Instance.new("Part")
        p.Name = "MynxxVizPath" -- protected from the FPS boost handlers
        p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CastShadow = false
        -- cylinder = rounded caps -> segments blend into each other (no square joints)
        p.Shape = Enum.PartType.Cylinder
        p.Material = Enum.Material.Neon; p.Color = color
        -- Feixe azul grosso, com brilho e transparência moderada.
        p.Size = Vector3.new(d.Magnitude, 2.4, 2.4)
        p.CFrame = CFrame.new((a + b) / 2, b) * CFrame.Angles(0, math.rad(90), 0)
        p.Transparency = 0.28
        p.Parent = workspace
        _vizParts[#_vizParts + 1] = p
        return p
    end
    local function vizDot(pos, color, sz)
        local p = Instance.new("Part")
        p.Name = "MynxxVizPath"
        p.Anchored = true; p.CanCollide = false; p.CanQuery = false; p.CastShadow = false
        p.Shape = Enum.PartType.Ball; p.Material = Enum.Material.Neon; p.Color = color
        p.Size = Vector3.new(sz, sz, sz)
        p.Position = pos
        _vizFadeIn(p, 0)
        p.Parent = workspace
        _vizParts[#_vizParts + 1] = p
        return p
    end
    -- Feixe único, grosso e azul conectando a origem ao destino.
    local function vizPath(fromPos, waypoints)
        clearViz(true)
        local destination = waypoints and waypoints[#waypoints]
        if not fromPos or not destination then return end
        if (destination - fromPos).Magnitude < 0.05 then return end
        vizLine(fromPos, destination, Color3.fromRGB(0, 170, 255))

        -- Bolas grandes como marcos da trajetória. O personagem passa
        -- rapidamente de um marco ao próximo, em vez de as bolas deslizarem.
        local points = { fromPos }
        local lastPoint = fromPos
        for _, point in ipairs(waypoints or {}) do
            if (point - lastPoint).Magnitude >= 14 then
                points[#points + 1] = point
                lastPoint = point
            end
        end
        if (destination - lastPoint).Magnitude >= 0.05 then
            points[#points + 1] = destination
        end
        local maxBalls = math.clamp(#points, 3, 8)
        for i = 1, maxBalls do
            local index = math.floor(1 + ((i - 1) * (#points - 1) / math.max(1, maxBalls - 1)) + 0.5)
            local point = points[math.clamp(index, 1, #points)]
            vizDot(point, Color3.fromRGB(255, 215, 0), 5.5)
        end
    end

    -- ===== Movement =====
    local SPEED = 200
    local ARRIVE = 3
    local MAX_CLIMB = 60

    local function vZero(hrp)
        if hrp then hrp.AssemblyLinearVelocity = Vector3.zero; hrp.AssemblyAngularVelocity = Vector3.zero end
    end

    -- Reactive obstacle dodge (same logic as the outer velMoveThrough copy)
    local _DODGE_SKIP = { ["DeliveryHitbox"]=true, ["StealHitbox"]=true, ["LaserHitbox"]=true,
        ["AnimalTarget"]=true, ["Multiplier"]=true, ["Laser"]=true, ["Hitbox"]=true,
        ["Spawn"]=true, ["MainRoot"]=true }
    local function _dodgeBlocked(origin, dirVec, dist)
        if dist <= 0.5 then return false end
        local rp = RaycastParams.new()
        rp.FilterType = Enum.RaycastFilterType.Exclude
        rp.IgnoreWater = true
        local skip = {}
        for _, pl in ipairs(Players:GetPlayers()) do
            if pl.Character then skip[#skip + 1] = pl.Character end
        end
        local u = dirVec.Unit
        local o = origin
        local remaining = dist
        for _ = 1, 8 do
            rp.FilterDescendantsInstances = skip
            local res = workspace:Raycast(o, u * remaining, rp)
            if not res then return false end
            local inst = res.Instance
            if inst and inst.CanCollide and not _DODGE_SKIP[inst.Name] then return true end
            remaining = remaining - ((res.Position - o).Magnitude + 0.3)
            if remaining <= 0 then return false end
            o = res.Position + u * 0.3
            skip[#skip + 1] = inst
        end
        return false
    end
    local _DODGE_UP = Vector3.new(0, 1, 0)
    -- Multi-ray check: cover shoulders + head/feet, not just a center ray
    local function _dodgePathBlocked(origin, dirVec, dist)
        local u = dirVec.Unit
        if _dodgeBlocked(origin, u, dist) then return true end
        local flat = Vector3.new(u.X, 0, u.Z)
        if flat.Magnitude > 0.01 then
            local perp = Vector3.new(-flat.Unit.Z, 0, flat.Unit.X) * 3
            if _dodgeBlocked(origin + perp, u, dist) or _dodgeBlocked(origin - perp, u, dist) then return true end
        end
        if _dodgeBlocked(origin + Vector3.new(0, 2.5, 0), u, dist)
            or _dodgeBlocked(origin - Vector3.new(0, 2.5, 0), u, dist) then return true end
        return false
    end
    local _lastDodgeVec, _lastDodgeUntil = nil, 0
    local function _dodgeSteer(origin, dir, dist)
        if not _dodgePathBlocked(origin, dir, dist) then
            if os.clock() > _lastDodgeUntil then _lastDodgeVec = nil end
            return dir
        end
        local flat = Vector3.new(dir.X, 0, dir.Z)
        if flat.Magnitude < 0.01 then return dir end
        flat = flat.Unit
        local perp = Vector3.new(-flat.Z, 0, flat.X)
        -- CLIMB-FIRST: keep the heading pointed AT the target and pass OVER the
        -- obstacle instead of orbiting sideways around it.
        local candidates = {
            (dir + _DODGE_UP * 1.25).Unit,
            (dir + _DODGE_UP * 2.5).Unit,
            (dir * 2 + perp + _DODGE_UP).Unit, (dir * 2 - perp + _DODGE_UP).Unit,
            (dir * 2 + perp * 2).Unit, (dir * 2 - perp * 2).Unit,
        }
        -- Commit to the same side briefly to avoid left/right flip-flop
        if _lastDodgeVec and os.clock() <= _lastDodgeUntil then
            table.insert(candidates, 1, _lastDodgeVec)
        end
        for _, cand in ipairs(candidates) do
            if not _dodgePathBlocked(origin, cand, dist) then
                _lastDodgeVec = cand; _lastDodgeUntil = os.clock() + 0.4
                return cand
            end
        end
        _lastDodgeVec = _DODGE_UP; _lastDodgeUntil = os.clock() + 0.4
        return _DODGE_UP -- fully boxed in: climb straight up
    end

    local function velMoveThrough(hrp, waypoints, speedOverride, allowJump, quickStart)
        if not hrp or not hrp.Parent or #waypoints == 0 then return end
        local _runSpeed = math.clamp(tonumber(speedOverride or MynxxSpeed.CARPET) or 450, 50, 900)
        vizPath(hrp.Position, waypoints)
        local wpIdx = 1
        local done = false
        local conn
        local function finish()
            if done then return end
            done = true
            if hrp and hrp.Parent then
                hrp.AssemblyLinearVelocity = Vector3.zero
                hrp.AssemblyAngularVelocity = Vector3.zero
                -- Only snap onto the final waypoint when we actually got close.
                -- On a stall/timeout far from the goal the old unconditional
                -- snap was a huge CFrame teleport -> anti-cheat lagback -> the
                -- caller believed we arrived and fired the clone from the
                -- wrong spot. Callers now verify distance and retry instead.
                local _fin = waypoints[#waypoints]
                if (hrp.Position - _fin).Magnitude <= 25 then
                    local _, y = hrp.CFrame:ToEulerAnglesYXZ()
                    hrp.CFrame = CFrame.new(_fin) * CFrame.Angles(0, y, 0)
                end
            end
            if conn then conn:Disconnect() end
            -- keep the path visible a moment so a failed TP shows WHERE it failed
            -- No vídeo, a linha some assim que o deslocamento termina.
            clearViz(true)
        end
        local lastDist, stall = math.huge, 0
        local lastEquip = 0
        if quickStart then
            local _hp = RaycastParams.new()
            _hp.FilterType = Enum.RaycastFilterType.Exclude
            _hp.IgnoreWater = true
            local _skip = {}
            for _, pl in ipairs(Players:GetPlayers()) do
                if pl.Character then _skip[#_skip + 1] = pl.Character end
            end
            _hp.FilterDescendantsInstances = _skip
            for _ = 1, 3 do
                local target = waypoints[wpIdx]
                if not target then break end
                local flat = Vector3.new(target.X - hrp.Position.X, 0, target.Z - hrp.Position.Z)
                local mag = flat.Magnitude
                if mag < 1 then break end
                local nextPos = hrp.Position + flat.Unit * math.min(35, mag)
                local _hit = workspace:Raycast(hrp.Position, nextPos - hrp.Position, _hp)
                if _hit and _hit.Instance and _hit.Instance.CanCollide then break end
                hrp.CFrame = (hrp.CFrame - hrp.CFrame.Position) + nextPos
                hrp.AssemblyLinearVelocity = Vector3.zero
                hrp.AssemblyAngularVelocity = Vector3.zero
                RunService.Heartbeat:Wait()
                if not hrp or not hrp.Parent then return end
            end
        end
        conn = RunService.Heartbeat:Connect(function()
            if not hrp or not hrp.Parent or done then
                if conn then conn:Disconnect() end; return
            end
            if os.clock() - lastEquip >= 0.25 then
                equipCarpet()
                lastEquip = os.clock()
            end
            local target = waypoints[wpIdx]
            local diff = target - hrp.Position
            local mag = diff.Magnitude
            if mag < ARRIVE then
                wpIdx = wpIdx + 1
                if wpIdx > #waypoints then finish(); return end
                lastDist, stall = math.huge, 0
                target = waypoints[wpIdx]
                diff = target - hrp.Position
                mag = diff.Magnitude
            end
            if mag > lastDist - 0.05 then stall = stall + 1 else stall = 0 end
            lastDist = mag
            if stall >= 18 then finish(); return end
            if mag >= 0.1 then
                local dir = diff.Unit
                if allowJump and diff.Y > 5 and wpIdx < #waypoints then
                    local hum = hrp.Parent and hrp.Parent:FindFirstChildOfClass("Humanoid")
                    if hum then
                        local st = hum:GetState()
                        if st ~= Enum.HumanoidStateType.Jumping and st ~= Enum.HumanoidStateType.Freefall then
                            pcall(function() hum:ChangeState(Enum.HumanoidStateType.Jumping) end)
                            pcall(function() hum.Jump = true end)
                        end
                    end
                end
                local _vy = dir.Y * _runSpeed
                if _vy > MAX_CLIMB then _vy = MAX_CLIMB end
                hrp.AssemblyLinearVelocity = Vector3.new(dir.X * _runSpeed, _vy, dir.Z * _runSpeed)
            end
        end)
        local totalDist = 0
        local prev = hrp.Position
        for _, wp in ipairs(waypoints) do totalDist = totalDist + (prev - wp).Magnitude; prev = wp end
        local timeout = math.max(2, totalDist / _runSpeed + 1.25)
        local elapsed = 0
        while not done and elapsed < timeout do task.wait(0.05); elapsed = elapsed + 0.05 end
        finish()
        vZero(hrp)
    end

    -- ===== Routing / Patch Finder =====
    local _DIRS = { Vector3.new(1,0,0), Vector3.new(-1,0,0), Vector3.new(0,0,1), Vector3.new(0,0,-1) }
    local _STRUCT = { ["structure base home"]=true, ["Wall"]=true, ["Floor"]=true, ["Roof"]=true }
    local _SKIP_NAME = { ["DeliveryHitbox"]=true, ["StealHitbox"]=true, ["LaserHitbox"]=true,
        ["AnimalTarget"]=true, ["Multiplier"]=true, ["Laser"]=true, ["Hitbox"]=true,
        ["Spawn"]=true, ["MainRoot"]=true, ["SecondFloor"]=true, ["ThirdFloor"]=true, ["Slope"]=true }
    -- Map center zone (CraftingMachine + new mid-map machines)
    local _MAP_CENTER = { minX = -458, maxX = -362, minZ = -40, maxZ = 185 }
    local _BYPASS_Z_NORTH, _BYPASS_Z_SOUTH = 205, -95
    local _BYPASS_X_WEST, _BYPASS_X_EAST = -525, -295
    local _CENTER_OBSTACLE_KEYWORDS = {
        "craftingmachine", "crafting", "forge", "fuse", "duel", "trade",
        "machine", "spin", "wheel", "event", "portal", "npc", "vendor",
    }
    local function _isMapObstacle(inst)
        if not inst then return false end
        local p = inst
        while p and p ~= workspace do
            local nl = p.Name:lower()
            for _, kw in ipairs(_CENTER_OBSTACLE_KEYWORDS) do
                if nl:find(kw, 1, true) then return true end
            end
            p = p.Parent
        end
        return false
    end
    local function _isConveyorPet(inst)
        -- moving conveyor brainrots are NOT route blockers: they pass through
        -- you and made straight diagonals randomly fail (L-shaped detours)
        return inst and inst:FindFirstAncestor("RenderedMovingAnimals") ~= nil
    end
    local function _blocks(inst)
        if not inst then return false end
        if _SKIP_NAME[inst.Name] then return false end
        if _isConveyorPet(inst) then return false end
        if _isMapObstacle(inst) then return true end
        if inst.CanCollide then return true end
        if _STRUCT[inst.Name] then return true end
        local s = inst.Size
        if s and math.max(s.X * s.Y, s.X * s.Z, s.Y * s.Z) > 150 then return true end
        return false
    end
    local function _blocksWide(inst)
        if not inst then return false end
        if _SKIP_NAME[inst.Name] then return false end
        if _isConveyorPet(inst) then return false end
        if _isMapObstacle(inst) then return true end
        if inst.CanCollide then return true end
        if _STRUCT[inst.Name] then return true end
        local s = inst.Size
        if s and math.max(s.X * s.Y, s.X * s.Z, s.Y * s.Z) > 30 then return true end
        return false
    end
    local function _block(origin, target, blockFn)
        blockFn = blockFn or _blocks
        local rp = RaycastParams.new(); rp.FilterType = Enum.RaycastFilterType.Exclude; rp.IgnoreWater = true
        local skip = {}
        for _, pl in ipairs(Players:GetPlayers()) do if pl.Character then skip[#skip + 1] = pl.Character end end
        local o = origin
        for _ = 1, 16 do
            rp.FilterDescendantsInstances = skip
            local d = target - o; if d.Magnitude < 0.05 then return nil end
            local res = workspace:Raycast(o, d, rp)
            if not res then return nil end
            if blockFn(res.Instance) then return res end
            skip[#skip + 1] = res.Instance; o = res.Position + d.Unit * 0.3
        end
        return nil
    end
    local function _clear(a, b) return _block(a, b) == nil end
    local function _clearDist(origin, dir, maxD)
        local res = _block(origin, origin + dir.Unit * maxD)
        if not res then return maxD end
        return (res.Position - origin).Magnitude
    end
    local function _pull(pts)
        if #pts <= 2 then return pts end
        local out = { pts[1] }; local i = 1
        while i < #pts do
            local j = #pts
            while j > i + 1 and not _clear(out[#out], pts[j]) do j = j - 1 end
            out[#out + 1] = pts[j]; i = j
        end
        return out
    end
    local function _clearWideRay(a, b) return _block(a, b, _blocksWide) == nil end
    local function _clearWide(a, b)
        if not _clear(a, b) then return false end
        local d = Vector3.new(b.X - a.X, 0, b.Z - a.Z)
        if d.Magnitude < 0.1 then return true end
        local _CLEARANCE = 6 -- 10 forced huge arcs, 4 dodged too late and clipped
        local perp = Vector3.new(-d.Z, 0, d.X).Unit * _CLEARANCE
        local up = Vector3.new(0, _CLEARANCE, 0)
        return _clearWideRay(a + perp, b + perp) and _clearWideRay(a - perp, b - perp)
            and _clearWideRay(a + up, b + up) and _clearWideRay(a - up, b - up)
    end
    local function _pullWide(pts)
        if #pts <= 2 then return pts end
        local out = { pts[1] }; local i = 1
        while i < #pts do
            local j = #pts
            while j > i + 1 and not _clearWide(out[#out], pts[j]) do j = j - 1 end
            out[#out + 1] = pts[j]; i = j
        end
        return out
    end
    local function _pushOffWalls(pts)
        if #pts <= 2 then return pts end
        local MARGIN = 7; local MAX_PUSH = 10 -- 10/14 pushed too far, 5/8 dodged too late
        local out = { pts[1] }
        for i = 2, #pts - 1 do
            local p = pts[i]; local shift = Vector3.zero
            for _, dr in ipairs(_DIRS) do
                local res = _block(p, p + dr * MARGIN, _blocks)
                if res then
                    local dist = (res.Position - p).Magnitude
                    if dist < MARGIN then shift = shift - dr * (MARGIN - dist) end
                end
            end
            if shift.Magnitude > 0.1 then
                if shift.Magnitude > MAX_PUSH then shift = shift.Unit * MAX_PUSH end
                local moved = p + shift
                if _clear(out[#out], moved) then out[#out + 1] = moved else out[#out + 1] = p end
            else
                out[#out + 1] = p
            end
        end
        out[#out + 1] = pts[#pts]
        return out
    end

    local function _inCenterZone(x, z)
        return x >= _MAP_CENTER.minX and x <= _MAP_CENTER.maxX
            and z >= _MAP_CENTER.minZ and z <= _MAP_CENTER.maxZ
    end
    local function _segmentCrossesCenter(a, b)
        if _inCenterZone(a.X, a.Z) or _inCenterZone(b.X, b.Z) then return true end
        for i = 1, 10 do
            local t = i / 11
            if _inCenterZone(a.X + (b.X - a.X) * t, a.Z + (b.Z - a.Z) * t) then return true end
        end
        return false
    end
    local function _findBestCenterDetour(fromPos, toPos, y, preferSideX)
        -- Side bypass (east/west) first = go AROUND center items, never through them
        local candidates = {
            { Vector3.new(_BYPASS_X_WEST, y, fromPos.Z), Vector3.new(_BYPASS_X_WEST, y, toPos.Z) },
            { Vector3.new(_BYPASS_X_EAST, y, fromPos.Z), Vector3.new(_BYPASS_X_EAST, y, toPos.Z) },
        }
        if preferSideX then
            table.insert(candidates, 1, {
                Vector3.new(preferSideX, y, fromPos.Z), Vector3.new(preferSideX, y, toPos.Z),
            })
        end
        for _, extra in ipairs({
            { Vector3.new(fromPos.X, y, _BYPASS_Z_NORTH), Vector3.new(toPos.X, y, _BYPASS_Z_NORTH) },
            { Vector3.new(fromPos.X, y, _BYPASS_Z_SOUTH), Vector3.new(toPos.X, y, _BYPASS_Z_SOUTH) },
            { Vector3.new(_BYPASS_X_WEST, y, fromPos.Z), Vector3.new(_BYPASS_X_WEST, y, toPos.Z) },
            { Vector3.new(_BYPASS_X_EAST, y, fromPos.Z), Vector3.new(_BYPASS_X_EAST, y, toPos.Z) },
            { Vector3.new(fromPos.X, y, _BYPASS_Z_NORTH), Vector3.new(toPos.X, y, toPos.Z) },
            { Vector3.new(fromPos.X, y, _BYPASS_Z_SOUTH), Vector3.new(toPos.X, y, toPos.Z) },
        }) do
            candidates[#candidates + 1] = extra
        end
        local best, bestLen = nil, math.huge
        for _, pair in ipairs(candidates) do
            local w1, w2 = pair[1], pair[2]
            if _clearWide(fromPos, w1) and _clearWide(w1, w2) and _clearWide(w2, toPos) then
                local len = (fromPos - w1).Magnitude + (w1 - w2).Magnitude + (w2 - toPos).Magnitude
                if len < bestLen then bestLen = len; best = { w1, w2 } end
            end
        end
        return best
    end
    local function _applyCenterPatch(fromPos, toPos, preferSideX)
        if not _segmentCrossesCenter(fromPos, toPos) then return nil end
        if _clearWide(fromPos, toPos) then return nil end
        return _findBestCenterDetour(fromPos, toPos, fromPos.Y, preferSideX)
    end
    local function _mergeRoute(prefix, route)
        if not prefix or #prefix == 0 then return route end
        if not route or #route == 0 then return prefix end
        local merged = {}
        for _, p in ipairs(prefix) do merged[#merged + 1] = p end
        local last = merged[#merged]
        for _, p in ipairs(route) do
            if (p - last).Magnitude > 2 then merged[#merged + 1] = p; last = p end
        end
        return merged
    end

    local PathfindingService = game:GetService("PathfindingService")
    local function computeRouteCore(fromPos, toPos, facingDir)
        if _clearWide(fromPos, toPos) then return { toPos } end
        -- (over-the-top hop removed: routes must stay grounded and weave
        -- BETWEEN center objects, never climb over them)
        local entry = facingDir and (toPos - facingDir * 14) or toPos
        local groundTo = Vector3.new(entry.X, fromPos.Y, entry.Z)
        local path = PathfindingService:CreatePath({
            -- AgentRadius 4 (was 12): pass ~3 studs beside objects, BETWEEN them
            AgentRadius = 4, AgentHeight = 5, AgentCanJump = true, AgentJumpHeight = 10, AgentMaxSlope = 89,
        })
        local FLOAT = 3
        local nav = { fromPos }
        local ok = pcall(function()
            path:ComputeAsync(Vector3.new(fromPos.X, fromPos.Y, fromPos.Z), groundTo)
        end)
        local pathOk = ok and path.Status == Enum.PathStatus.Success
        if not pathOk then
            -- pathfinding failed (start inside a base = off the navmesh):
            -- retry from the ground point right below the start
            local ok2 = pcall(function()
                local rp = RaycastParams.new()
                rp.FilterType = Enum.RaycastFilterType.Exclude
                local ex = {}
                for _, pl in ipairs(Players:GetPlayers()) do
                    if pl.Character then ex[#ex + 1] = pl.Character end
                end
                rp.FilterDescendantsInstances = ex
                local res = workspace:Raycast(fromPos, Vector3.new(0, -80, 0), rp)
                local start2 = res and (res.Position + Vector3.new(0, 2, 0)) or fromPos
                path:ComputeAsync(start2, groundTo)
            end)
            pathOk = ok2 and path.Status == Enum.PathStatus.Success
        end
        if pathOk then
            local last = fromPos
            for _, wp in ipairs(path:GetWaypoints()) do
                if (wp.Position - last).Magnitude >= 8 then
                    nav[#nav + 1] = wp.Position + Vector3.new(0, FLOAT, 0); last = wp.Position
                end
            end
        else
            -- STILL no path: never fall back to a raw straight line THROUGH the
            -- bases -- route along the street corridors instead
            local det = _findBestCenterDetour(fromPos, toPos, 4, nil)
                or _findBestCenterDetour(fromPos, toPos, 10, nil)
            if det then
                for _, w in ipairs(det) do nav[#nav + 1] = w end
            end
        end
        nav[#nav + 1] = entry + Vector3.new(0, FLOAT, 0)
        nav = _pushOffWalls(nav)
        local route = _pullWide(nav)
        route[#route + 1] = toPos
        return route
    end
    local function computeRoute(fromPos, toPos, facingDir)
        -- SIMPLE: one pathfinding curve that weaves BETWEEN the center objects
        -- (~3 studs beside them), glued to the ground. No corridor bypass, no
        -- offset dodges, no over-the-top -- the curve IS the avoidance.
        local flatDist = Vector3.new(toPos.X - fromPos.X, 0, toPos.Z - fromPos.Z).Magnitude
        local route = computeRouteCore(fromPos, toPos, facingDir)
        route = _pullWide(_pushOffWalls(route))
        -- FLOOR-2 CENTER AVOID: the ground curve weaves BETWEEN the center
        -- objects (~3 studs beside them), which is fine flat -- but a floor-2
        -- target ramps the Y up along the way, so that same XZ line ends up
        -- skimming the Trade Plaza roof, grazes it and the flight stalls.
        -- When the destination is HIGHER and the curve crosses the center
        -- zone, reroute it AROUND the center via the street corridors (the
        -- red-line path): the continuous climb then happens over open street,
        -- never over the plaza.
        if (toPos.Y - fromPos.Y) > 8 then
            local crosses = false
            local prev = fromPos
            for _, p in ipairs(route) do
                if _segmentCrossesCenter(prev, p) then crosses = true; break end
                prev = p
            end
            if crosses then
                local det = _findBestCenterDetour(fromPos, toPos, fromPos.Y, nil)
                    or _findBestCenterDetour(fromPos, toPos, 4, nil)
                    or _findBestCenterDetour(fromPos, toPos, 10, nil)
                if det then
                    local re = {}
                    for _, w in ipairs(det) do re[#re + 1] = w end
                    re[#re + 1] = toPos
                    route = re
                end
            end
        end
        -- GROUND CLAMP: outside the target base the WHOLE route hugs the REAL
        -- floor (raycast down under each waypoint, fly 3 studs above it) -- for
        -- floor-1 AND floor-2 targets alike, neighboring base included. Same
        -- curve, same corners -- just glued to the ground, so obstacles are
        -- passed BESIDE, never over. Only the FINAL point keeps its height
        -- (the climb onto the podium).
        -- EXCEPTION: short hop (adjacent base) to FLOOR 2 -- clamping there left
        -- no room to climb and the flight slammed into the base. Skip the clamp
        -- and ramp up early instead.
        local shortHighHop = flatDist < 140 and (toPos.Y - fromPos.Y) > 8
        if flatDist > 60 and not shortHighHop then
            local rp = RaycastParams.new()
            rp.FilterType = Enum.RaycastFilterType.Exclude
            local ex = {}
            for _, pl in ipairs(Players:GetPlayers()) do
                if pl.Character then ex[#ex + 1] = pl.Character end
            end
            rp.FilterDescendantsInstances = ex
            for i = 1, #route - 1 do
                local p = route[i]
                local res = workspace:Raycast(Vector3.new(p.X, p.Y + 4, p.Z), Vector3.new(0, -80, 0), rp)
                local gY = res and (res.Position.Y + 3) or 3
                if p.Y > gY then route[i] = Vector3.new(p.X, gY, p.Z) end
            end
        end
        -- ROUTE POLISH: (1) drop collinear waypoints so straights are ONE clean
        -- segment, then (2) round the REAL corners only (fixed ~6-stud radius).
        -- Rounding every waypoint turned the trace into a broken bead chain.
        if #route >= 3 then
            local simp = { route[1] }
            for i = 2, #route - 1 do
                local a, b, c = simp[#simp], route[i], route[i + 1]
                local d1 = Vector3.new(b.X - a.X, 0, b.Z - a.Z)
                local d2 = Vector3.new(c.X - b.X, 0, c.Z - b.Z)
                if d1.Magnitude > 0.5 and d2.Magnitude > 0.5 and d1.Unit:Dot(d2.Unit) < 0.99 then
                    simp[#simp + 1] = b -- keep only real direction changes
                end
            end
            simp[#simp + 1] = route[#route]
            route = simp
            if #route >= 3 then
                local sm = { route[1] }
                for i = 2, #route - 1 do
                    local a, b, c = route[i - 1], route[i], route[i + 1]
                    local la, lc = (b - a).Magnitude, (b - c).Magnitude
                    sm[#sm + 1] = b:Lerp(a, math.min(0.35, la > 0.1 and 6 / la or 0))
                    sm[#sm + 1] = b:Lerp(c, math.min(0.35, lc > 0.1 and 6 / lc or 0))
                end
                sm[#sm + 1] = route[#route]
                route = sm
            end
        end
        -- SHORT HOP TO FLOOR 2: climb IMMEDIATELY -- full podium height must be
        -- reached by ~half the trip, way before the base wall (MAX_CLIMB caps
        -- vertical speed, so a late climb slams into the structure).
        -- CLIMB PROFILE for a higher target: ONE straight diagonal TOWARD the
        -- brainrot -- the climb is spread over the whole horizontal distance
        -- (no vertical spike at the start, no time wasted), capped at podium
        -- height (never overshoots, so it can never descend into the base).
        if (toPos.Y - fromPos.Y) > 8 and #route >= 2 then
            local targetY = toPos.Y + 3
            local total, prev = 0, fromPos
            local cum = {}
            for i = 1, #route do
                total = total + math.sqrt((route[i].X - prev.X) ^ 2 + (route[i].Z - prev.Z) ^ 2)
                cum[i] = total
                prev = route[i]
            end
            if total < 1 then total = 1 end
            for i = 1, #route - 1 do
                local p = route[i]
                local wantY = fromPos.Y + (targetY - fromPos.Y) * (cum[i] / total)
                local y = math.max(p.Y, wantY)
                if y > targetY then y = targetY end
                route[i] = Vector3.new(p.X, y, p.Z)
            end
            -- monotonic: the diagonal only ever goes UP
            local maxY
            for i = 1, #route - 1 do
                local p = route[i]
                if maxY and p.Y < maxY then route[i] = Vector3.new(p.X, maxY, p.Z)
                else maxY = p.Y end
            end
        end
        -- BASE CLEARANCE (final pass): every intermediate waypoint stays at
        -- least 3 studs OUTSIDE any base that isn't the start or target base.
        -- The curve slides along the wall then re-aims at the target once past.
        do
            local plots = workspace:FindFirstChild("Plots")
            if plots and #route >= 1 then
                local PAD = 3
                local boxes = {}
                local targetBox = nil
                for _, plot in ipairs(plots:GetChildren()) do
                    pcall(function()
                        local cf, size = plot:GetBoundingBox()
                        local hx, hz = size.X / 2, size.Z / 2
                        local px, pz = cf.Position.X, cf.Position.Z
                        local fromIn = math.abs(fromPos.X - px) < hx and math.abs(fromPos.Z - pz) < hz
                        local toIn = math.abs(toPos.X - px) < hx and math.abs(toPos.Z - pz) < hz
                        if not fromIn and not toIn then
                            boxes[#boxes + 1] = { px = px, pz = pz, hx = hx, hz = hz }
                        elseif toIn and not fromIn then
                            -- TARGET base: early/mid waypoints must stay OUT of it
                            -- too (neighbor-base TPs were clipping its walls on
                            -- approach) -- only the final stretch may enter
                            targetBox = { px = px, pz = pz, hx = hx, hz = hz }
                        end
                    end)
                end
                local function pushOut(p, b)
                    local dx, dz = p.X - b.px, p.Z - b.pz
                    if math.abs(dx) < b.hx + PAD and math.abs(dz) < b.hz + PAD then
                        local escX = b.hx + PAD - math.abs(dx)
                        local escZ = b.hz + PAD - math.abs(dz)
                        if escX <= escZ then
                            return Vector3.new(b.px + (dx >= 0 and 1 or -1) * (b.hx + PAD), p.Y, p.Z)
                        else
                            return Vector3.new(p.X, p.Y, b.pz + (dz >= 0 and 1 or -1) * (b.hz + PAD))
                        end
                    end
                    return p
                end
                if #route >= 3 then
                    local lastOutside = math.floor((#route - 1) * 0.7)
                    for i = 2, #route - 1 do
                        local p = route[i]
                        for _, b in ipairs(boxes) do p = pushOut(p, b) end
                        if targetBox and i <= lastOutside then p = pushOut(p, targetBox) end
                        route[i] = p
                    end
                end
                -- SEGMENT PASS: pushing WAYPOINTS out is not enough -- the straight
                -- LEG between two valid points can still cut the corner of a base
                -- in between (adjacent-base TP kept grazing the first neighbour,
                -- stalling the flight and forcing a huge re-route). Clip every leg
                -- against each foreign base box in 2D and, when it crosses, insert
                -- corner waypoints ~6 studs off the walls that go AROUND it.
                if #boxes > 0 then
                    local function segHitsBox(a, b, bx)
                        -- 2D slab test vs the box expanded by PAD-0.5 (points that
                        -- pushOut placed exactly at PAD stay strictly outside)
                        local ex, ez = bx.hx + PAD - 0.5, bx.hz + PAD - 0.5
                        local dx, dz = b.X - a.X, b.Z - a.Z
                        local ox, oz = a.X - bx.px, a.Z - bx.pz
                        local t0, t1 = 0, 1
                        if math.abs(dx) < 1e-6 then
                            if math.abs(ox) >= ex then return false end
                        else
                            local ta, tb = (-ex - ox) / dx, (ex - ox) / dx
                            if ta > tb then ta, tb = tb, ta end
                            t0 = math.max(t0, ta); t1 = math.min(t1, tb)
                            if t0 > t1 then return false end
                        end
                        if math.abs(dz) < 1e-6 then
                            if math.abs(oz) >= ez then return false end
                        else
                            local ta, tb = (-ez - oz) / dz, (ez - oz) / dz
                            if ta > tb then ta, tb = tb, ta end
                            t0 = math.max(t0, ta); t1 = math.min(t1, tb)
                            if t0 > t1 then return false end
                        end
                        return true
                    end
                    local function detourAround(a, b, bx)
                        -- corner candidates sit at PAD+3 so the flight (ARRIVE
                        -- radius cuts corners by ~3 studs) still clears the wall
                        local ex, ez = bx.hx + PAD + 3, bx.hz + PAD + 3
                        local corners = {
                            { bx.px + ex, bx.pz + ez }, { bx.px + ex, bx.pz - ez },
                            { bx.px - ex, bx.pz + ez }, { bx.px - ex, bx.pz - ez },
                        }
                        local best, bestLen = nil, nil
                        for _, c in ipairs(corners) do
                            local cp = Vector3.new(c[1], (a.Y + b.Y) / 2, c[2])
                            if not segHitsBox(a, cp, bx) and not segHitsBox(cp, b, bx) then
                                local len = (a - cp).Magnitude + (cp - b).Magnitude
                                if not bestLen or len < bestLen then bestLen, best = len, { cp } end
                            end
                        end
                        if best then return best end
                        -- no single corner works: pass along one full side (2 corners)
                        for ci = 1, 4 do
                            for cj = 1, 4 do
                                if ci ~= cj then
                                    local c1 = Vector3.new(corners[ci][1], a.Y, corners[ci][2])
                                    local c2 = Vector3.new(corners[cj][1], b.Y, corners[cj][2])
                                    if not segHitsBox(a, c1, bx) and not segHitsBox(c1, c2, bx)
                                        and not segHitsBox(c2, b, bx) then
                                        local len = (a - c1).Magnitude + (c1 - c2).Magnitude + (c2 - b).Magnitude
                                        if not bestLen or len < bestLen then bestLen, best = len, { c1, c2 } end
                                    end
                                end
                            end
                        end
                        return best
                    end
                    -- include the START position so the very first leg is checked
                    -- too (a "clear" straight shot can dive through a base's open
                    -- front that raycasts never catch)
                    local full = { fromPos }
                    for _, p in ipairs(route) do full[#full + 1] = p end
                    local rebuilt = { full[1] }
                    for i = 2, #full do
                        local pending = { full[i] }
                        local guard = 0
                        while #pending > 0 and guard < 16 do
                            guard = guard + 1
                            local a = rebuilt[#rebuilt]
                            local nxt = pending[1]
                            local hitBox = nil
                            for _, bx in ipairs(boxes) do
                                if segHitsBox(a, nxt, bx) then hitBox = bx; break end
                            end
                            local det = hitBox and detourAround(a, nxt, hitBox)
                            if det then
                                for k = #det, 1, -1 do table.insert(pending, 1, det[k]) end
                            else
                                rebuilt[#rebuilt + 1] = nxt
                                table.remove(pending, 1)
                            end
                        end
                        for _, rem in ipairs(pending) do rebuilt[#rebuilt + 1] = rem end
                    end
                    table.remove(rebuilt, 1) -- drop the injected start position
                    route = rebuilt
                end
            end
        end
        return route
    end

    -- ===== Clone / goToBrainrot =====
    local function doClone()
        if not NetModule then pcall(loadNet) end
        local char = LP.Character or LP.CharacterAdded:Wait()
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not char or not hum then return false end
        local cloner = (LP:FindFirstChild("Backpack") and LP.Backpack:FindFirstChild("Quantum Cloner"))
                    or char:FindFirstChild("Quantum Cloner")
        if not cloner then return false end
        if cloner.Parent ~= char then
            pcall(function() hum:EquipTool(cloner) end)
            task.wait()
        end
        if not NetModule then return false end
        local useOk = pcall(function() NetModule:RemoteEvent("UseItem"):FireServer() end)
        task.wait(0.05)
        local telOk = pcall(function() NetModule:RemoteEvent("QuantumCloner/OnTeleport"):FireServer() end)
        return useOk and telOk
    end

    -- CARPET GLIDE: fly straight to the pet on the equipped carpet at FULL 3D speed
    -- (same pace as velMoveThrough -- WalkTPSpeed in the actual direction, vertical
    -- included). The carpet is already equipped by goToBrainrot, so no extra equip
    -- wait. Used as the FINAL approach after the clone/grabble TP.
    local function carpetGlideTo(targetPos)
        if not targetPos then return end
        local char = LP.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        pcall(function() hrp.Anchored = false end)
        pcall(equipCarpet)
        local SPEED = (Config.TpSettings and (tonumber(Config.TpSettings.WalkTPSpeed) or tonumber(Config.TpSettings.GrabbleTPSpeed))) or 190
        local climbY = getGoToBrainrotTargetY(targetPos.Y)
        local floatOffset = (targetPos.Y > 20) and -4 or 0
        local midGoal = Vector3.new(targetPos.X, climbY, targetPos.Z)
        local finalGoal = Vector3.new(targetPos.X, targetPos.Y + floatOffset, targetPos.Z)

        local function glideTo(goal, timeout)
            local t0 = os.clock()
            local lastDist, stall = math.huge, 0
            local lastEquip = 0
            while hrp.Parent and (os.clock() - t0) < (timeout or 8) do
                if LP:GetAttribute("Stealing") then break end
                if os.clock() - lastEquip >= 0.25 then
                    equipCarpet()
                    lastEquip = os.clock()
                end
                local diff = goal - hrp.Position
                local mag = diff.Magnitude
                if mag < 3 then break end
                if mag > lastDist - 0.05 then stall = stall + 1 else stall = 0 end
                lastDist = mag
                if stall >= 30 then break end
                hrp.AssemblyLinearVelocity = diff.Unit * SPEED
                RunService.Heartbeat:Wait()
            end
        end

        -- Same pattern all floors: climb to floor walk height at pet XZ, then final glide to pet
        if math.abs(hrp.Position.Y - climbY) > 2 or (hrp.Position - midGoal).Magnitude > 4 then
            glideTo(midGoal, 8)
        end
        if (hrp.Position - finalGoal).Magnitude > 3 then
            glideTo(finalGoal, 6)
        end

        if hrp and hrp.Parent then
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
        end
    end

    local function goToBrainrot(petPos)
        if not petPos then return end
        local char, hrp, hum
        local _t0 = os.clock()
        repeat
            char = LP.Character
            hrp = char and char:FindFirstChild("HumanoidRootPart")
            hum = char and char:FindFirstChildOfClass("Humanoid")
            if hrp and hum then break end
            RunService.Heartbeat:Wait()
        until os.clock() - _t0 > 3
        if not hrp or not hum then return end
        pcall(function() hrp.Anchored = false end)
        local _equipped = false
        do
            local _e0 = os.clock()
            repeat
                char = LP.Character
                for _, _cn in ipairs(CARPET_NAMES) do
                    if char and char:FindFirstChild(_cn) then _equipped = true; break end
                end
                if _equipped then break end
                equipCarpet()
                RunService.Heartbeat:Wait()
            until _equipped or os.clock() - _e0 > 1.5
            if _equipped then RunService.Heartbeat:Wait() end
        end
        char = LP.Character
        hrp = char and char:FindFirstChild("HumanoidRootPart")
        hum = char and char:FindFirstChildOfClass("Humanoid")
        if not hrp then return end
        pcall(function() hrp.Anchored = false end)
        local _plotRad = (petPos.Y <= 8.9) and 26 or 25
        do
            local _t0b = os.clock()
            repeat
                local p = hrp.Position
                local inRad = false
                local plotsFolder = workspace:FindFirstChild("Plots")
                if plotsFolder then
                    for _, plot in ipairs(plotsFolder:GetChildren()) do
                        pcall(function()
                            local pp = plot:GetPivot().Position
                            if math.abs(p.X - pp.X) < _plotRad and math.abs(p.Z - pp.Z) < _plotRad then inRad = true end
                        end)
                        if inRad then break end
                    end
                end
                if inRad then break end
                RunService.Heartbeat:Wait()
            until os.clock() - _t0b > 1.5
        end
        local h = petPos.Y
        local targetY = getGoToBrainrotTargetY(h)
        local _to = Vector3.new(petPos.X, targetY, petPos.Z)
        -- keep the go-to-brainrot trace on screen for 5s (clearViz defers)
        _vizStickyUntil = os.clock() + 5
        if Config.TpSettings and Config.TpSettings.BrainrotCarpet then
            vizPath(hrp.Position, { petPos }) -- show the path in carpet mode too
            carpetGlideTo(petPos)
            clearViz()
        else
            local _noclip = {}
            pcall(function()
                for _, _p in ipairs(char:GetDescendants()) do
                    if _p:IsA("BasePart") then
                        _noclip[#_noclip+1] = { p = _p, v = _p.CanCollide }
                        _p.CanCollide = false
                    end
                end
            end)
            local _route = computeRoute(hrp.Position, _to, nil)
            if not _route or #_route == 0 then _route = { _to } end
            velMoveThrough(hrp, _route, (Config and Config.TpSettings and (tonumber(Config.TpSettings.WalkTPSpeed) or tonumber(Config.TpSettings.GrabbleTPSpeed))) or MynxxSpeed.INBASE, true, true)
            for _, _r in ipairs(_noclip) do
                pcall(function() _r.p.CanCollide = _r.v end)
            end
        end
        if hrp and hrp.Parent then
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
        end
        do
            local _platY = getGoToBrainrotTargetY(petPos.Y) - 1.5
            local _plat = Instance.new("Part")
            _plat.Name = "MynxxTempPlatform"; _plat.Size = Vector3.new(8, 1, 8)
            _plat.Position = Vector3.new(petPos.X, _platY, petPos.Z)
            _plat.Anchored = true; _plat.CanCollide = false; pcall(makeOneWay, _plat); _plat.Transparency = 1
            _plat.Material = Enum.Material.SmoothPlastic; _plat.Parent = workspace
            task.spawn(function()
                local _s = tick()
                while tick() - _s < 20 do
                    if LP:GetAttribute("Stealing") then break end
                    task.wait(0.1)
                end
                if _plat and _plat.Parent then _plat:Destroy() end
            end)
        end
    end

    -- ===== Steal engine =====
    local InternalStealCache = {}
    local STEAL_HOLD_DURATION = 1.3
    local _stealHoldStart = 0
    local _stealHoldActive = false

    local function buildStealCallbacks(prompt)
        if InternalStealCache[prompt] then return end
        if not prompt or not prompt.Parent then return end
        local data = { holdCallbacks = {}, triggerCallbacks = {}, holdEndCallbacks = {}, ready = true }
        local function grab(sig, into)
            local ok, conns = pcall(getconnections, sig)
            if ok and type(conns) == "table" then
                for _, c in ipairs(conns) do
                    if type(c.Function) == "function" then table.insert(into, c.Function) end
                end
            end
        end
        grab(prompt.PromptButtonHoldBegan, data.holdCallbacks)
        grab(prompt.Triggered, data.triggerCallbacks)
        grab(prompt.PromptButtonHoldEnded, data.holdEndCallbacks)
        if #data.holdCallbacks > 0 or #data.triggerCallbacks > 0 or #data.holdEndCallbacks > 0 then
            InternalStealCache[prompt] = data
        end
    end

    local function executeStealAsync(prompt)
        local data = InternalStealCache[prompt]
        if not data or not data.ready then return false end
        data.ready = false
        _stealHoldStart = tick()
        _stealHoldActive = true
        _G.Mynxx_StealStatus = _G.Mynxx_StealStatus or {}
        _G.Mynxx_StealStatus.active = true
        _G.Mynxx_StealStatus.start = _stealHoldStart
        _G.Mynxx_StealStatus.duration = STEAL_HOLD_DURATION

        task.spawn(function()
            for _, fn in ipairs(data.holdCallbacks) do task.spawn(fn) end
            pcall(function()
                local _st = prompt:GetAttribute("State")
                if _st ~= nil and _st ~= "Steal" then
                    if not _G._xenStealRemote then
                        local _net = _G.Net
                        _G._xenStealRemote = _net and _net:RemoteEvent("f40f7d9e-2f0d-4167-b250-899273f46874")
                    end
                    local r = _G._xenStealRemote
                    if r then
                        local _t = workspace:GetServerTimeNow() + 124
                        r:FireServer(_t, "68c86eb7-eb7e-4b4d-96ae-cf7cd847c5b0")
                        r:FireServer(_t, "07b9cc25-2a1f-4a26-a0ec-f2fab578d8bd")
                    end
                end
            end)
            local remain = STEAL_HOLD_DURATION - (tick() - _stealHoldStart)
            if remain > 0 then task.wait(remain) end
            if prompt and prompt.Parent then
                for _, fn in ipairs(data.triggerCallbacks) do task.spawn(fn) end
            end
            for _, fn in ipairs(data.holdEndCallbacks) do task.spawn(fn) end
            _stealHoldActive = false
            if _G.Mynxx_StealStatus then _G.Mynxx_StealStatus.active = false end
            task.wait(0.05)
            data.ready = true
        end)
        return true
    end

    local function findStealPrompt(pet)
        if pet.plot and pet.slot then
            local plots = workspace:FindFirstChild("Plots")
            local plot = plots and plots:FindFirstChild(pet.plot)
            local podiums = plot and plot:FindFirstChild("AnimalPodiums")
            local podium = podiums and podiums:FindFirstChild(tostring(pet.slot))
            if podium then
                local base = podium:FindFirstChild("Base")
                local spawn = base and base:FindFirstChild("Spawn")
                local attach = spawn and spawn:FindFirstChild("PromptAttachment")
                if attach then
                    for _, p in ipairs(attach:GetChildren()) do
                        if p:IsA("ProximityPrompt") then return p end
                    end
                end
                for _, d in ipairs(podium:GetDescendants()) do
                    if d:IsA("ProximityPrompt") then return d end
                end
            end
        end
        if pet.model and pet.model.Parent then
            for _, d in ipairs(pet.model:GetDescendants()) do
                if d:IsA("ProximityPrompt") then return d end
            end
        end
        return nil
    end

    local STEAL_PROXIMITY = 60
    local STEAL_ARM_TIMEOUT = 25
    local _stealTarget = nil
    local _stealArmedAt = 0

    local function timeUntilCanSteal()
        if LP:GetAttribute("Stealing") or LP:GetAttribute("IsTrading")
            or LP:GetAttribute("IsDuelSelecting") or LP:GetAttribute("Web") then
            return -1
        end
        return 0
    end

    local function _petKey(pet)
        if not pet then return nil end
        if pet.plot and pet.slot then return tostring(pet.plot) .. "|" .. tostring(pet.slot) end
        return "idx|" .. tostring(pet.index or pet.name)
    end
    local function _petStillExists(sel, pets)
        if not sel or not pets then return nil end
        local k = _petKey(sel)
        for _, p in ipairs(pets) do if _petKey(p) == k then return p end end
        return nil
    end
    local function _pickByMode(pets)
        if not pets or #pets == 0 then return nil end
        local C = Config or {}
        -- PRIORITY FIRST -- ALWAYS, whatever the mode (same rule as the outer
        -- getTargetPetData: "priority pets before anything else"). It used to be
        -- gated behind Auto-TP Priority / Steal Priority: in Highest mode a
        -- priority brainrot on the CONVEYOR lost to any base pet with more gen
        -- and the TP flew to the base instead of the belt. Follow the user's
        -- priorityList order (1 -> 2 -> 3 ...), match by DisplayName OR Index,
        -- and take the TOPMOST priority entry present on the server (conveyor
        -- included), regardless of distance/gen.
        if priorityList and #priorityList > 0 then
            for _, pName in ipairs(priorityList) do
                local searchName = pName:lower()
                for _, p in ipairs(pets) do
                    if (p.name and p.name:lower() == searchName) or (p.index and p.index:lower() == searchName) then
                        return p
                    end
                end
            end
        end
        -- HIGHEST gen/value: Auto-TP Highest Gen/Value OR Steal "Highest".
        if C.AutoTPHighestGen or C.AutoTPHighestValue or (C.StealMode == "Highest") then
            local best, bv
            for _, p in ipairs(pets) do
                local v = p.mps or 0
                if not bv or v > bv then bv, best = v, p end
            end
            return best
        end
        -- Auto TP Priority fallback: pega o de maior $/s se não tiver priority
        if C.AutoTPPriority then
            local best, bv
            for _, p in ipairs(pets) do
                local v = p.mps or 0
                if not bv or v > bv then bv, best = v, p end
            end
            if best then return best end
        end
        return pets[1]
    end
    local function _samePet(a, b)
        if not a or not b then return false end
        return a.plot == b.plot and tostring(a.slot) == tostring(b.slot)
    end

    local _stealTarget2 = nil
    local function armSteal(pet)
        if not pet then return end
        _stealTarget = pet; _stealArmedAt = os.clock()
        _stealTarget2 = pet
        _G.Mynxx_StealStatus = _G.Mynxx_StealStatus or {}
        _G.Mynxx_StealStatus.target = pet
    end
    local function disarmSteal()
        _stealTarget = nil
        _G.Mynxx_StealStatus = _G.Mynxx_StealStatus or {}
        _G.Mynxx_StealStatus.target = nil
        _G.Mynxx_StealStatus.active = false
    end
    _G.MynxxArmSteal = armSteal
    _G.MynxxDisarmSteal = disarmSteal

    local AUTO_STEAL = (Config and Config.AutoStealEnabled) and true or false
    _G.MynxxAutoSteal = function(on) AUTO_STEAL = on ~= false end

    local _stealLastScan = 0
    local _autoLastScan = 0
    local isTeleporting = false
    local _tpStartedAt = 0
    local _cloneTP = false
    local _cloneFired = false
    local _started = false

    RunService.Heartbeat:Connect(function()
        local now = os.clock()
        if AUTO_STEAL and _started and not LP:GetAttribute("Stealing")
            and not (isTeleporting and _cloneTP)
            and (now - _autoLastScan) >= ((isTeleporting or _G._isTpMoving) and 0.75 or 0.1) then
            _autoLastScan = now
            local char = LP.Character
            local hrp = char and char:FindFirstChild("HumanoidRootPart")
            if hrp then
                local ok, pets = pcall(scanAllPets)
                if ok and pets then
                    local best
                    -- Manual selection from MynxxHub
                    if manuallySelectedUID then
                        for _, pet in ipairs(pets) do
                            if not pet.conveyor and pet.plot and pet.slot then
                                local uid = pet.plot .. "_" .. tostring(pet.slot)
                                if uid == manuallySelectedUID then best = pet; break end
                            end
                        end
                    end
                    if not best then
                        local STEAL_MODE = (Config and Config.StealMode) or "Priority"
                        if STEAL_MODE == "Nearest" then
                            local bd
                            for _, pet in ipairs(pets) do
                                if not pet.conveyor then
                                    local prompt = findStealPrompt(pet)
                                    if prompt and prompt.Parent then
                                        local pp = prompt.Parent
                                        local ppPos = (pp:IsA("BasePart") and pp.Position)
                                            or (pp.Parent and pp.Parent:IsA("BasePart") and pp.Parent.Position)
                                        if ppPos then
                                            local d = (hrp.Position - ppPos).Magnitude
                                            if (not bd or d < bd) then bd, best = d, pet end
                                        end
                                    end
                                end
                            end
                        else
                            local nc = {}
                            for _, p in ipairs(pets) do if not p.conveyor then nc[#nc + 1] = p end end
                            best = _pickByMode(nc)
                        end
                    end
                    if best then
                        if not _samePet(best, _stealTarget) then armSteal(best) end
                    elseif _stealTarget and not _stealHoldActive then
                        disarmSteal()
                    end
                end
            end
        end

        local pet = _stealTarget
        if not pet then return end
        local STEAL_MODE = (Config and Config.StealMode) or "Priority"
        if STEAL_MODE == "Nearest" and now - _stealArmedAt > STEAL_ARM_TIMEOUT then
            disarmSteal(); return
        end
        if now - _stealLastScan < 0.067 then return end
        _stealLastScan = now
        local t = timeUntilCanSteal()
        if t == -1 then
            if LP:GetAttribute("Stealing") then disarmSteal() end
            return
        end
        if t > 0 and t > STEAL_HOLD_DURATION then return end
        local char = LP.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then return end
        local prompt = findStealPrompt(pet)
        if not prompt or not prompt.Parent then return end
        local pp = prompt.Parent
        local ppPos = (pp and pp:IsA("BasePart") and pp.Position)
            or (pp and pp.Parent and pp.Parent:IsA("BasePart") and pp.Parent.Position)
        local _inCloneTP = isTeleporting and _cloneTP and _cloneFired
        if not _inCloneTP and ppPos and (hrp.Position - ppPos).Magnitude > STEAL_PROXIMITY then return end
        local oldMax
        pcall(function() oldMax = prompt.MaxActivationDistance end)
        pcall(function() prompt.MaxActivationDistance = math.huge end)
        buildStealCallbacks(prompt)
        if InternalStealCache[prompt] then executeStealAsync(prompt) end
        pcall(function() if oldMax ~= nil then prompt.MaxActivationDistance = oldMax end end)
    end)

    -- ===== Main TP function (1:1 xentp.lua doVelocityTP) =====
    local function doVelocityTP()
        if isTeleporting and (os.clock() - _tpStartedAt) < 20 then return end
        isTeleporting = true
        _G._isTpMoving = true
        _tpStartedAt = os.clock()
        _cloneTP = false
        _cloneFired = false
        clearViz(true)

        if not NetModule then pcall(loadNet) end

        local char = LP.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not hrp or not hum then isTeleporting = false; _G._isTpMoving = false; return end

        local allPets = scanAllPets()
        if #allPets == 0 then
            local _t0 = os.clock()
            while #allPets == 0 and os.clock() - _t0 < 2.5 do
                task.wait(0.05)
                allPets = scanAllPets()
            end
        end
        if #allPets == 0 then isTeleporting = false; _G._isTpMoving = false; return end

        local pet
        if manuallySelectedUID then
            for _, p in ipairs(allPets) do
                if p.plot and p.slot and (p.plot .. "_" .. tostring(p.slot)) == manuallySelectedUID then pet = p; break end
            end
        end
        if not pet then pet = _pickByMode(allPets) or allPets[1] end

        local _tpSpd = (Config and Config.TpSettings and Config.TpSettings.GrabbleTPSpeed) or 500
        local _cloneDelay = (Config and Config.TpSettings and Config.TpSettings.CloneDelayVal) or 0.25

        local petPos = pet.position
        local petName = pet.name

        local _diedDuringTP = false
        local _diedConn = hum.Died:Connect(function() _diedDuringTP = true end)

        local adjY = petPos.Y
        if TALL_PETS[petName] then adjY = petPos.Y - TALL_OFFSET end
        local coordTable = getCloneCoordTable(adjY)

        if pet.conveyor then
            local model = pet.model
            local maxHP = hum.MaxHealth
            hum.Health = maxHP
            local healConn = RunService.Heartbeat:Connect(function()
                if hum and hum.Parent then hum.Health = maxHP end
            end)
            carpetEngage()
            vZero(hrp)
            local function livePos()
                if not model or not model.Parent then return nil end
                local part = model.PrimaryPart or model:FindFirstChildWhichIsA("BasePart")
                if part then return part.Position end
                local ok, piv = pcall(function() return model:GetPivot().Position end)
                return ok and piv or nil
            end
            local _t0 = os.clock()
            while os.clock() - _t0 < 6 do
                if _diedDuringTP then break end
                if not hrp or not hrp.Parent then break end
                local lp = livePos()
                if not lp then break end
                local diff = lp - hrp.Position
                if diff.Magnitude <= 5 then break end
                equipCarpet()
                hrp.AssemblyLinearVelocity = diff.Unit * _tpSpd
                RunService.Heartbeat:Wait()
            end
            if hrp and hrp.Parent then
                hrp.AssemblyLinearVelocity = Vector3.zero
                hrp.AssemblyAngularVelocity = Vector3.zero
            end
            healConn:Disconnect()
            if _diedConn then _diedConn:Disconnect() end
            isTeleporting = false
            _G._isTpMoving = false
            return
        end

        _cloneTP = true
        disarmSteal()

        local closestData, skyKey = findClosest(petPos, coordTable)
        if not closestData or not skyKey then
            if _diedConn then _diedConn:Disconnect() end
            isTeleporting = false; _G._isTpMoving = false; return
        end

        local destPos = closestData.coord
        local maxHP = hum.MaxHealth
        hum.Health = maxHP
        local healConn = RunService.Heartbeat:Connect(function()
            if hum and hum.Parent then hum.Health = maxHP end
        end)

        carpetEngage()
        vZero(hrp)

        local facingDir = closestData.facing == "NORTH" and Vector3.new(0, 0, -1) or Vector3.new(0, 0, 1)
        do
            local isUpper = (coordTable == UPPER)
            local idx = getClosestBaseIdx(petPos)
            local frontCoord, frontFace = buildFrontCandidate(idx, isUpper, hrp.Position.Z)
            local sides = plotSides(coordTable, idx)
            local baseRef = BASES_LOW[idx]
            local flatToBase = math.sqrt((hrp.Position.X - baseRef.X) ^ 2 + (hrp.Position.Z - baseRef.Z) ^ 2)
            local sameColumn = (hrp.Position.X < COLUMN_SPLIT_X) == (idx <= 4)
            if sameColumn and flatToBase < 150 and #sides > 0 then
                local bs, bd
                for _, d in ipairs(sides) do
                    local dd = (hrp.Position - d.coord).Magnitude
                    if not bd or dd < bd then bd, bs = dd, d end
                end
                destPos = bs.coord
                facingDir = bs.facing == "NORTH" and Vector3.new(0, 0, -1) or Vector3.new(0, 0, 1)
            else
                local bestCoord, bestFace = frontCoord, frontFace
                local bestDist = (hrp.Position - frontCoord).Magnitude
                for _, d in ipairs(sides) do
                    local dd = (hrp.Position - d.coord).Magnitude
                    if dd < bestDist then
                        bestDist = dd
                        bestCoord = d.coord
                        bestFace = d.facing == "NORTH" and Vector3.new(0, 0, -1) or Vector3.new(0, 0, 1)
                    end
                end
                destPos = bestCoord
                facingDir = bestFace
            end
        end

        if hrp and hrp.Parent then
            hrp.CFrame = CFrame.new(hrp.Position, hrp.Position + facingDir)
            hrp.AssemblyAngularVelocity = Vector3.zero
        end

        -- O vídeo mostra um deslocamento único e retilíneo, sem waypoints
        -- visíveis nem desvios laterais. A validação de chegada permanece
        -- abaixo para evitar considerar uma tentativa interrompida como sucesso.
        local _route = { destPos }
        local _stepped = {}
        do
            local startY = hrp.Position.Y
            local destY = destPos.Y
            local prev = hrp.Position
            local totalFlat = 0
            for _, wp in ipairs(_route) do
                totalFlat = totalFlat + (Vector3.new(wp.X, 0, wp.Z) - Vector3.new(prev.X, 0, prev.Z)).Magnitude
                prev = wp
            end
            if totalFlat < 0.01 then totalFlat = 0.01 end
            local SEG = 40
            prev = hrp.Position
            local travelled = 0
            for _, wp in ipairs(_route) do
                local flatVec = Vector3.new(wp.X, 0, wp.Z) - Vector3.new(prev.X, 0, prev.Z)
                local legFlat = flatVec.Magnitude
                if legFlat >= 0.01 then
                    local subs = math.max(1, math.ceil(legFlat / SEG))
                    for s = 1, subs do
                        local f = s / subs
                        local px = prev.X + (wp.X - prev.X) * f
                        local pz = prev.Z + (wp.Z - prev.Z) * f
                        local along = travelled + legFlat * f
                        local rampY = startY + (destY - startY) * (along / totalFlat)
                        _stepped[#_stepped + 1] = Vector3.new(px, rampY, pz)
                    end
                else
                    _stepped[#_stepped + 1] = wp
                end
                travelled = travelled + legFlat
                prev = wp
            end
            if #_stepped > 0 then _stepped[#_stepped] = _route[#_route] end
        end

        -- Sem quickStart: evita pequenos saltos de CFrame e mantém o deslize
        -- contínuo, semelhante ao movimento mostrado no vídeo.
        velMoveThrough(hrp, _stepped, _tpSpd, true, false)

        do
            local _rc = LP.Character
            hrp = _rc and _rc:FindFirstChild("HumanoidRootPart")
            hum = _rc and _rc:FindFirstChildOfClass("Humanoid")
            if _diedDuringTP or not hrp or not hum or (hrp.Position - destPos).Magnitude > 18 then
                if _diedConn then _diedConn:Disconnect() end
                healConn:Disconnect()
                if hrp and hrp.Parent then vZero(hrp) end
                isTeleporting = false
                _G._isTpMoving = false
                return
            end
        end

        hrp.CFrame = CFrame.new(destPos, destPos + facingDir)
        vZero(hrp)

        local syncFrames = 3
        local syncConn
        syncConn = RunService.Heartbeat:Connect(function()
            if not hrp or not hrp.Parent then syncConn:Disconnect(); return end
            syncFrames = syncFrames - 1
            hrp.CFrame = CFrame.new(destPos, destPos + facingDir)
            hrp.AssemblyLinearVelocity = Vector3.zero
            hrp.AssemblyAngularVelocity = Vector3.zero
            if syncFrames <= 0 then syncConn:Disconnect() end
        end)

        task.wait(0.2)

        healConn:Disconnect()

        do
            local _chk = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
            if _diedDuringTP or not _chk or not _chk.Parent or (_chk.Position - destPos).Magnitude > 20 then
                if _diedConn then _diedConn:Disconnect() end
                if _chk and _chk.Parent then vZero(_chk) end
                isTeleporting = false
                _G._isTpMoving = false
                return
            end
        end

        armSteal(pet)
        _cloneFired = true

        local _ahrp = LP.Character and LP.Character:FindFirstChild("HumanoidRootPart")
        local _clonePos = (_ahrp and _ahrp.Parent and _ahrp.Position) or destPos

        if _ahrp and _ahrp.Parent then
            _ahrp.AssemblyLinearVelocity = Vector3.zero
            _ahrp.AssemblyAngularVelocity = Vector3.zero
            pcall(function() _ahrp.Anchored = true end)
            task.delay(0.5, function()
                if _ahrp and _ahrp.Parent then pcall(function() _ahrp.Anchored = false end) end
            end)
        end

        local _charAdded = false
        local _caConn = LP.CharacterAdded:Connect(function() _charAdded = true end)

        task.wait(_cloneDelay)
        local _cloneOk = pcall(instantClone)

        if _cloneOk then
            local _cloneSucceeded = false
            local _t0 = os.clock()
            local plotsFolder = workspace:FindFirstChild("Plots")
            while not _cloneSucceeded and (os.clock() - _t0) < 1.5 do
                task.wait(0.03)
                if _diedDuringTP then break end
                local _c = LP.Character
                local _h = _c and _c:FindFirstChild("HumanoidRootPart")
                if _h and _h.Parent then
                    local p = _h.Position
                    if (p - _clonePos).Magnitude > 30 then
                        _cloneSucceeded = true
                    elseif plotsFolder then
                        local _rad = (petPos.Y <= 8.9) and 30 or 30
                        for _, plot in ipairs(plotsFolder:GetChildren()) do
                            pcall(function()
                                local pp = plot:GetPivot().Position
                                if math.abs(p.X - pp.X) < _rad and math.abs(p.Z - pp.Z) < _rad then
                                    _cloneSucceeded = true
                                end
                            end)
                            if _cloneSucceeded then break end
                        end
                    end
                end
            end
            if _caConn then _caConn:Disconnect() end
            if _cloneSucceeded and not _diedDuringTP then
                task.wait(0.5)
                goToBrainrot(petPos)
            end
        else
            if _caConn then _caConn:Disconnect() end
        end

        if _diedConn then _diedConn:Disconnect() end
        isTeleporting = false
        _G._isTpMoving = false
    end

    doGrabbleVelocityTP = doVelocityTP
    _G.MynxxStartSideTP = doVelocityTP

    _G.Mynxx_ExecuteManualTP = function()
        task.spawn(function() pcall(doVelocityTP) end)
    end

    UIS.InputBegan:Connect(function(input, gameProcessed)
        if gameProcessed then return end
        if input.KeyCode == Enum.KeyCode.T then
            task.spawn(function() pcall(doVelocityTP) end)
        end
    end)

-- ============================================================
-- ANTI BODY SWAP DE POSIÇÃO (SXE Remasterd) - VERSÃO MELHORADA
-- Detecta troca de lugar com outro jogador e troca de volta
-- ============================================================
-- EXPORTA VARIÁVEL GLOBAL PARA CONTROLE
_G.antiSwapEnabled = true  -- ativado por padrão

_G.setAntiSwapState = function(state)
    _G.antiSwapEnabled = state == true
    pcall(ShowNotification, "ANTI-SWAP", _G.antiSwapEnabled and "ATIVADO" or "DESATIVADO")
end

_G.toggleAntiSwap = function()
    _G.setAntiSwapState(not _G.antiSwapEnabled)
end

task.spawn(function()
    local cooldown = 0
    local COOLDOWN_TIME = 2.5
    local lastPosition = nil
    local lastPlayerPositions = {}
    local wasStealing = false
    local stealCheckDelay = 0.5  -- tempo para verificar se perdeu o pet

    local function findPlayerAtPosition(pos, margin)
        margin = margin or 2.5
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer then
                local char = plr.Character
                if char then
                    local hrp = char:FindFirstChild("HumanoidRootPart")
                    if hrp and (hrp.Position - pos).Magnitude <= margin then
                        return plr
                    end
                end
            end
        end
        return nil
    end

    local function performSwap(targetPlayer)
        if not targetPlayer then return false end
        if tick() - cooldown < COOLDOWN_TIME then return false end
        cooldown = tick()

        local char = LocalPlayer.Character
        local hum = char and char:FindFirstChildOfClass("Humanoid")
        if not hum then return false end

        -- 1) Tenta encontrar o item "Body Swap Potion"
        local swapItem = LocalPlayer.Backpack:FindFirstChild("Body Swap Potion") 
                        or char:FindFirstChild("Body Swap Potion")
        
        if swapItem then
            hum:EquipTool(swapItem)
            task.wait(0.1)
            swapItem:Activate()
            -- Tenta ativar no alvo via RemoteClick
            local remote = swapItem:FindFirstChild("RemoteClick")
            if remote then 
                remote:FireServer(targetPlayer)
            else
                pcall(function() swapItem:Activate() end)
            end
            ShowNotification("ANTI-SWAP", "Trocando de lugar com " .. targetPlayer.Name)
            return true
        end

        -- 2) Fallback: UseItem RemoteEvent
        local useRemote = _G.Net and _G.Net:RemoteEvent("UseItem")
        if useRemote then
            pcall(function() 
                useRemote:FireServer(12, targetPlayer)
            end)
            ShowNotification("ANTI-SWAP", "Swap ativado via UseItem")
            return true
        end

        -- 3) Procura qualquer remote com "body" ou "swap" no nome
        local foundRemote = nil
        for _, v in ipairs(ReplicatedStorage:GetDescendants()) do
            if v:IsA("RemoteEvent") then
                local name = v.Name:lower()
                if name:find("body") or name:find("swap") then
                    foundRemote = v
                    break
                end
            end
        end
        if foundRemote then
            pcall(function() foundRemote:FireServer(targetPlayer) end)
            ShowNotification("ANTI-SWAP", "Swap ativado via " .. foundRemote.Name)
            return true
        end

        ShowNotification("ANTI-SWAP", "Item/remote de swap não encontrado!")
        return false
    end

    -- Monitora o atributo Stealing
    LocalPlayer:GetAttributeChangedSignal("Stealing"):Connect(function()
        local isStealing = LocalPlayer:GetAttribute("Stealing") == true
        
        -- Se estava roubando e parou de roubar
        if wasStealing and not isStealing then
            -- Verifica se houve teleporte recente e alguém está na sua posição antiga
            if lastPosition then
                local char = LocalPlayer.Character
                local hrp = char and char:FindFirstChild("HumanoidRootPart")
                if hrp then
                    local dist = (hrp.Position - lastPosition).Magnitude
                    if dist > 25 then
                        local offender = findPlayerAtPosition(lastPosition, 4)
                        if offender then
                            local wasThereBefore = false
                            local oldPos = lastPlayerPositions[offender.UserId]
                            if oldPos and (oldPos - hrp.Position).Magnitude <= 4 then
                                wasThereBefore = true
                            end
                            if wasThereBefore then
                                ShowNotification("ANTI-SWAP", "Swap detectado durante steal com " .. offender.Name)
                                task.wait(0.2)
                                performSwap(offender)
                            end
                        end
                    end
                end
            end
        end
        wasStealing = isStealing
    end)

    -- Loop principal de monitoramento de posição
    while true do
        task.wait(0.1)
        if not _G.antiSwapEnabled then continue end

        local char = LocalPlayer.Character
        local hrp = char and char:FindFirstChild("HumanoidRootPart")
        if not hrp then 
            lastPosition = nil
            continue 
        end

        local currentPos = hrp.Position
        if not lastPosition then
            lastPosition = currentPos
            for _, plr in ipairs(Players:GetPlayers()) do
                if plr ~= LocalPlayer then
                    local pChar = plr.Character
                    local pHrp = pChar and pChar:FindFirstChild("HumanoidRootPart")
                    if pHrp then
                        lastPlayerPositions[plr.UserId] = pHrp.Position
                    end
                end
            end
            continue
        end

        local dist = (currentPos - lastPosition).Magnitude
        if dist > 25 then
            local offender = findPlayerAtPosition(lastPosition, 4)
            if offender then
                local wasThereBefore = false
                local oldPos = lastPlayerPositions[offender.UserId]
                if oldPos and (oldPos - currentPos).Magnitude <= 4 then
                    wasThereBefore = true
                end
                if wasThereBefore then
                    -- Se não estiver roubando, detecta normalmente
                    if not LocalPlayer:GetAttribute("Stealing") then
                        ShowNotification("ANTI-SWAP", "Swap detectado com " .. offender.Name)
                        task.wait(0.2)
                        performSwap(offender)
                    end
                end
            end
        end

        lastPosition = currentPos
        for _, plr in ipairs(Players:GetPlayers()) do
            if plr ~= LocalPlayer then
                local pChar = plr.Character
                local pHrp = pChar and pChar:FindFirstChild("HumanoidRootPart")
                if pHrp then
                    lastPlayerPositions[plr.UserId] = pHrp.Position
                end
            end
        end
    end
end)
-- ============================================================
-- ============================================================
-- ============================================================

    -- Boot
    task.spawn(function() pcall(loadModules) pcall(loadNet) end)
    task.spawn(function()
        -- Scanning does NOT need the character (the Heartbeat loop checks hrp
        -- itself), so start probing immediately instead of waiting for spawn.
        pcall(loadModules); pcall(loadNet)
        local _t0 = os.clock()
        local lastCount, streak = -1, 0
        repeat
            local ok, pets, misses = pcall(scanAllPets)
            if ok and pets then
                local n = #pets
                if n == lastCount then streak = streak + 1 else streak = 0; lastCount = n end
                -- Start as soon as a scan is COMPLETE (no unsynced/unrendered
                -- entries) and confirmed by a second identical scan -- this is
                -- what prevents launching on a partial scan that misses brainrots.
                -- If something stays permanently unresolvable, a count stable for
                -- ~1s is good enough (old behavior started on ANY non-empty scan).
                if n > 0 and (((misses or 0) == 0 and streak >= 1) or streak >= 6) then break end
            end
            task.wait(0.05)
        until os.clock() - _t0 > 12
        _started = true
        _G.MynxxScannerReady = true -- TpOnLoad fast path waits on this
    end)
end
