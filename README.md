--[[
    [SYSTEM OUTPUT] Rayfield UI Integration (5-in-1 Ultimate Suite)
    [DESCRIPTION] 요청하신 5가지 기능(Death Point, Singularity V7, Auto Equip, Save Instance, Auto Kill/Kill Aura)을 
                  Rayfield UI에 완벽히 통합하고 토글식으로 온오프 가능하게 최적화한 최종 스크립트입니다.
--]]

local Rayfield = loadstring(game:HttpGet('https://githubusercontent.com'))()

-- 윈도우 생성
local Window = Rayfield:CreateWindow({
   Name = "Ultimate Automation Hub",
   LoadingTitle = "Rayfield Interface Suite",
   LoadingSubtitle = "Multi-Functional Script",
   ConfigurationSaving = {
      Enabled = true,
      FolderName = "UltimateConfig", 
      FileName = "MainHub"
   },
   Discord = {
      Enabled = false,
      Invite = "sirius",
      RememberJoins = true
   },
   KeySystem = true,
   KeySettings = {
      Title = "Sirius Hub",
      Subtitle = "Key System",
      Note = "Join the discord (discord.gg/sirius)",
      FileName = "SiriusKey",
      SaveKey = true,
      GrabKeyFromSite = false,
      Key = "Hello" -- 키 시스템 암호: Hello
   }
})

--------------------------------------------------------------------------------
-- [SECTION 1] 공통 환경 변수 및 초기화
--------------------------------------------------------------------------------
local game = game
local workspace = workspace
local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local localPlayer = Players.LocalPlayer

local isA = game.IsA
local findFirstChildWhichIsA = game.FindFirstChildWhichIsA
local findFirstChild = game.FindFirstChild
local spawnThread = task.spawn
local deferThread = task.defer

--------------------------------------------------------------------------------
-- [SECTION 2] Core 1 & 2: Singularity V7 & Death Point 관련 로직
--------------------------------------------------------------------------------
local _CHARACTER, _HUMANOID, _ROOTPART, _HANDLE
local _RAW_ITEMS, _ITEM_COUNT, _CURRENT_ITEM

if _G.SingularityEngine then
    _G.SingularityEngine.Active = false
    for _, signal in ipairs(_G.SingularityEngine.Signals) do
        if signal then signal:Disconnect() end
    end
end

_G.SingularityEngine = { Active = false, Signals = {} }
local singularityRegistry = _G.SingularityEngine.Signals

local function quantumVortex(item)
    if not _G.SingularityEngine.Active then return end
    if not item or item.Parent ~= workspace then return end

    if isA(item, "Tool") or isA(item, "BackpackItem") then
        _CHARACTER = localPlayer.Character
        if _CHARACTER and _CHARACTER.Parent then
            _HUMANOID = findFirstChildWhichIsA(_CHARACTER, "Humanoid")
            if _HUMANOID and _HUMANOID.Health > 0 then
                _ROOTPART = _CHARACTER.PrimaryPart or findFirstChild(_CHARACTER, "HumanoidRootPart")
                spawnThread(function()
                    pcall(function()
                        _HANDLE = findFirstChild(item, "Handle") or findFirstChildWhichIsA(item, "BasePart")
                        if _HANDLE and _ROOTPART then
                            _HANDLE.CFrame = _ROOTPART.CFrame
                            _HANDLE.AssemblyLinearVelocity = _ROOTPART.AssemblyLinearVelocity
                        end
                        _HUMANOID:EquipTool(item)
                    end)
                end)
            end
        end
    end
end

local function staticGlobalClean()
    if not _G.SingularityEngine or not _G.SingularityEngine.Active then return end
    pcall(function()
        _RAW_ITEMS = workspace:GetChildren()
        _ITEM_COUNT = #_RAW_ITEMS
        for i = 1, _ITEM_COUNT do
            _CURRENT_ITEM = _RAW_ITEMS[i]
            if _CURRENT_ITEM then quantumVortex(_CURRENT_ITEM) end
        end
    end)
end

local function startSingularityEngine()
    _G.SingularityEngine.Active = true
    singularityRegistry[#singularityRegistry + 1] = workspace.DescendantAdded:Connect(quantumVortex)
    singularityRegistry[#singularityRegistry + 1] = localPlayer.CharacterAdded:Connect(function(char)
        deferThread(function()
            local humanoid = char:WaitForChild("Humanoid", 2)
            if humanoid then staticGlobalClean() end
        end)
    end)
    singularityRegistry[#singularityRegistry + 1] = RunService.RenderStepped:Connect(staticGlobalClean)
    singularityRegistry[#singularityRegistry + 1] = RunService.Heartbeat:Connect(staticGlobalClean)
    singularityRegistry[#singularityRegistry + 1] = RunService.Stepped:Connect(staticGlobalClean)
    singularityRegistry[#singularityRegistry + 1] = RunService.PostSimulation:Connect(staticGlobalClean)

    spawnThread(function()
        local state = _G.SingularityEngine
        while state.Active do
            RunService.PreSimulation:Wait()
            staticGlobalClean()
        end
    end)
end

local function stopSingularityEngine()
    _G.SingularityEngine.Active = false
    for i, signal in ipairs(singularityRegistry) do
        if signal then signal:Disconnect() end
        singularityRegistry[i] = nil
    end
end

-- Death Point 로직
local isDeathActive = false
local deathPosition = nil
local deathConnection = nil

local function enableDeathPoint()
    if isDeathActive then return end
    isDeathActive = true

    local function onCharacterAdded(character)
        local rootPart = character:WaitForChild("HumanoidRootPart", 10)
        local humanoid = character:WaitForChild("Humanoid", 10)
        if humanoid and rootPart then
            humanoid.Died:Connect(function()
                if not isDeathActive then return end
                deathPosition = rootPart.Position
                localPlayer.CharacterAdded:Wait()
                local newCharacter = localPlayer.Character or localPlayer.CharacterAdded:Wait()
                local newRootPart = newCharacter:WaitForChild("HumanoidRootPart", 10)
                if newRootPart and deathPosition then
                    newRootPart.CFrame = CFrame.new(deathPosition)
                end
            end)
        end
    end

    if localPlayer.Character then onCharacterAdded(localPlayer.Character) end
    deathConnection = localPlayer.CharacterAdded:Connect(onCharacterAdded)
end

local function disableDeathPoint()
    isDeathActive = false
    deathPosition = nil
    if deathConnection then
        deathConnection:Disconnect()
        deathConnection = nil
    end
end

--------------------------------------------------------------------------------
-- [SECTION 3] Core 3: Auto Equip (자동 툴 장착 기능) 로직
--------------------------------------------------------------------------------
local isAutoEquipActive = false
local autoEquipConnections = {}
local autoEquipHumanoid = nil
local equippedTools = {}

local function autoEquipTool(tool)
    if not isAutoEquipActive or not autoEquipHumanoid then return end
    if equippedTools[tool] then return end
    equippedTools[tool] = true
    pcall(function()
        autoEquipHumanoid:EquipTool(tool)
    end)
end

local function onAutoEquipChild(tool)
    if tool.ClassName == "Tool" then
        autoEquipTool(tool)
    end
end

local function onAutoEquipCharacter(char)
    table.clear(equippedTools)
    autoEquipHumanoid = char:WaitForChild("Humanoid")
    
    local children = workspace:GetChildren()
    for i = 1, #children do
        local v = children[i]
        if v.ClassName == "Tool" then
            deferThread(autoEquipTool, v)
        end
    end
end

local function enableAutoEquip()
    if isAutoEquipActive then return end
    isAutoEquipActive = true
    if localPlayer.Character then onAutoEquipCharacter(localPlayer.Character) end
    autoEquipConnections[#autoEquipConnections + 1] = workspace.ChildAdded:Connect(onAutoEquipChild)
    autoEquipConnections[#autoEquipConnections + 1] = localPlayer.CharacterAdded:Connect(onAutoEquipCharacter)
end

local function disableAutoEquip()
    isAutoEquipActive = false
    autoEquipHumanoid = nil
    table.clear(equippedTools)
    for i, connection in ipairs(autoEquipConnections) do
        if connection then connection:Disconnect() end
        autoEquipConnections[i] = nil
    end
end

--------------------------------------------------------------------------------
-- [SECTION 4] Core 5: Kill Aura / Auto Kill (친구 제외 타격) 로직
--------------------------------------------------------------------------------
local isKillAuraActive = false
local killAuraConnection = nil
local killAuraRange = 1000 -- 슬라이더 연동 기본값

local function isFriendWith(player1, player2)
    return player1:IsFriendsWith(player2.UserId)
end

local function activateToolOnCharacter(tool, character)
    if tool and tool:FindFirstChild("Handle") then
        tool:Activate()
        for _, part in ipairs(character:GetChildren()) do
            if part:IsA("BasePart") then
                firetouchinterest(tool.Handle, part, 0)
                firetouchinterest(tool.Handle, part, 1)
            end
        end
    end
end

local function isWithinRange(player1, player2)
    local character1 = player1.Character
    local character2 = player2.Character
    if character1 and character2 then
        local root1 = character1:FindFirstChild("HumanoidRootPart")
        local root2 = character2:FindFirstChild("HumanoidRootPart")
        if root1 and root2 then
            return (root1.Position - root2.Position).magnitude <= killAuraRange
        end
    end
    return false
end

local function startKillAura()
    if isKillAuraActive then return end
    isKillAuraActive = true
    
    killAuraConnection = RunService.RenderStepped:Connect(function()
        if not isKillAuraActive then return end
        local players = Players:GetPlayers()
        for i = 1, #players do
            local otherPlayer = players[i]
            if otherPlayer ~= localPlayer and not isFriendWith(localPlayer, otherPlayer) and isWithinRange(localPlayer, otherPlayer) then
                local character = otherPlayer.Character
                if character then
local tool = localPlayer.Character and localPlayer.Character:FindFirstChildOfClass("Tool")if tool thenactivateToolOnCharacter(tool, character)endendendendend)endlocal function stopKillAura()isKillAuraActive = falseif killAuraConnection thenkillAuraConnection:Disconnect()killAuraConnection = nilendend

-- [SECTION 5] Rayfield UI 디자인 및 컴포넌트 맵핑local AutomationTab = Window:CreateTab("Automation", 4483362458)local CombatTab = Window:CreateTab("Combat", 4483362458)local MapSaveTab = Window:CreateTab("Map Saver", 4483362458)-- 1. Automation Tab 컴포넌트AutomationTab:CreateSection("Utility & Item Cheats")AutomationTab:CreateToggle({Name = "Enable Death Point",CurrentValue = false,Flag = "DeathPointToggle",Callback = function(Value)if Value thenenableDeathPoint()Rayfield:Notify({Title = "Death Point Active",
Content = "죽었던 위치로 자동 리스폰 텔레포트합니다.", Duration = 3, Image = 4483362458})elsedisableDeathPoint()Rayfield:Notify({Title = "Death Point Deactivated", Content = "기능이 해제되었습니다.", Duration = 3, Image = 4483362458})endend,})AutomationTab:CreateToggle({Name = "🔱 Singularity V7 (Auto Loot)",CurrentValue = false,Flag = "SingularityToggle",Callback = function(Value)if Value thenstartSingularityEngine()Rayfield:Notify({Title = "🔱 SINGULARITY V7", Content = "FINAL OMNIPOTENCE ENGINE ACTIVE", Duration = 4, Image = 4483362458})else
stopSingularityEngine()Rayfield:Notify({Title = "🔱 SINGULARITY V7", Content = "ENGINE DISABLED", Duration = 3, Image = 4483362458})endend,})AutomationTab:CreateToggle({Name = "🔧 Auto Equip Tools",CurrentValue = false,Flag = "AutoEquipToggle",Callback = function(Value)if Value thenenableAutoEquip()Rayfield:Notify({Title = "Auto Equip Active", Content = "바닥에 생성되는 도구를 즉시 자동 장착합니다.", Duration = 3, Image = 4483362458})else
disableAutoEquip()Rayfield:Notify({Title = "Auto Equip Deactivated", Content = "도구 자동 장착 기능이 꺼졌습니다.", Duration = 3, Image = 4483362458})endend,})-- 2. Combat Tab 컴포넌트 (추가된 Kill Aura)CombatTab:CreateSection("Combat Automation")CombatTab:CreateToggle({Name = "⚔️ Kill Aura (Auto Attack)",CurrentValue = false,Flag = "KillAuraToggle",Callback = function(Value)if Value thenstartKillAura()
Rayfield:Notify({Title = "Kill Aura Active", Content = "범위 내의 적(친구 제외)을 자동으로 공격합니다.", Duration = 3, Image = 4483362458})elsestopKillAura()Rayfield:Notify({Title = "Kill Aura Deactivated", Content = "킬 아우라 기능이 꺼졌습니다.", Duration = 3, Image = 4483362458})endend,})CombatTab:CreateSlider({Name = "🎯 Attack Range (거리 설정)",Range = {10, 2000},Increment = 50,Suffix = "Studs",CurrentValue = 1000,Flag = "KillRangeSlider",Callback = function(Value)killAuraRange = Value
end,})-- 3. Map Save Tab 컴포넌트MapSaveTab:CreateSection("Universal Instance Saver")MapSaveTab:CreateButton({Name = "💾 Run SynSaveInstance (Save Place)",Callback = function()Rayfield:Notify({Title = "Save Instance", Content = "맵 데이터 추출을 시작합니다. 잠시 멈춤 현상이 있을 수 있습니다.", Duration = 4, Image = 4483362458})pcall(function()local Params = {RepoURL = "githubusercontent.com",SSI = "saveinstance",}local synsaveinstance = loadstring(game:HttpGet(Params.RepoURL .. Params.SSI .. ".luau", true), Params.SSI)()local Options = {}synsaveinstance(Options)Rayfield:Notify({Title = "Save Success", Content = "추출이 완료되었습니다. workspace 폴더를 확인하세요.", Duration = 4, Image = 4483362458})end)end,})local InfoParagraph = AutomationTab:CreateParagraph({Title = "Notice",Content = "모든 기능은 독립적으로 작동하며 비활성화 시 내부 RenderStepped 연결 및 루프가 완전히 해제됩니다."})
