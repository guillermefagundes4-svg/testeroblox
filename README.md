--[[
    Script de Automação para Seu Jogo
    Versão Melhorada e Otimizada
    Autor: Seu Amigo (com melhorias)
--]]

-- ========== CONFIGURAÇÕES ==========
local CONFIG = {
    LoopDelay = 0.3,           -- Delay entre cada ciclo (segundos)
    EnableAutoRoll = true,     -- Ativar rolagem automática de máquinas
    EnableAutoSell = true,     -- Ativar venda automática
    EnableAutoUpgrade = true,  -- Ativar upgrades automáticos
    EnableAutoFloor = true,    -- Ativar compra de andares
    EnableAutoEquip = true,    -- Ativar equipar melhores máquinas
    DebugMode = false,         -- Modo debug (exibir logs)
}

-- ========== INICIALIZAÇÃO ==========
local Players = game:GetService("Players")
local ReplicatedStorage = game:GetService("ReplicatedStorage")
local LocalPlayer = Players.LocalPlayer

-- Cache de referências para melhor performance
local ClientToServer = ReplicatedStorage:WaitForChild("ClientToServer")
local RemoteFunctions = {
    RollMachine = ClientToServer:WaitForChild("RollMachine"),
    EquipBestMachines = ClientToServer:WaitForChild("EquipBestMachines"),
    BuyFloor = ClientToServer:WaitForChild("BuyFloor"),
    GetAvailableUpgrades = ReplicatedStorage:WaitForChild("ClientToServer"):WaitForChild("GetAvailableUpgrades"),
    BuyUpgrade = ReplicatedStorage:WaitForChild("ClientToServer"):WaitForChild("BuyUpgrade"),
}

local RemoteEvents = {
    SellStorage = ClientToServer:WaitForChild("SellStorage"),
}

-- ========== FUNÇÕES AUXILIARES ==========
local function Log(message)
    if CONFIG.DebugMode then
        print("[AutoFarm]: " .. message)
    end
end

local function SafeInvoke(remoteFunction, ...)
    local success, result = pcall(function()
        return remoteFunction:InvokeServer(...)
    end)
    
    if success then
        return result
    else
        Log("Erro ao chamar " .. remoteFunction.Name .. ": " .. tostring(result))
        return nil
    end
end

local function SafeFire(remoteEvent, ...)
    local success, result = pcall(function()
        remoteEvent:FireServer(...)
    end)
    
    if not success then
        Log("Erro ao disparar " .. remoteEvent.Name .. ": " .. tostring(result))
    end
end

-- ========== FUNÇÕES DE AUTOMAÇÃO ==========
local Automation = {}

function Automation.RollMachines()
    if not CONFIG.EnableAutoRoll then return end
    SafeInvoke(RemoteFunctions.RollMachine)
    Log("Máquinas roladas")
end

function Automation.SellStorage()
    if not CONFIG.EnableAutoSell then return end
    SafeFire(RemoteEvents.SellStorage)
    Log("Armazenamento vendido")
end

function Automation.EquipBest()
    if not CONFIG.EnableAutoEquip then return end
    SafeInvoke(RemoteFunctions.EquipBestMachines)
    Log("Melhores máquinas equipadas")
end

function Automation.BuyFloor()
    if not CONFIG.EnableAutoFloor then return end
    SafeInvoke(RemoteFunctions.BuyFloor)
    Log("Tentativa de comprar andar")
end

function Automation.CheckAndBuyUpgrades()
    if not CONFIG.EnableAutoUpgrade then return end
    local upgrades = SafeInvoke(RemoteFunctions.GetAvailableUpgrades)
    
    if upgrades then
        local boughtCount = 0
        for upgradeId, status in pairs(upgrades) do
            if status == "AVAILABLE" then
                SafeInvoke(RemoteFunctions.BuyUpgrade, upgradeId)
                boughtCount += 1
                Log("Upgrade comprado: " .. tostring(upgradeId))
                task.wait(0.1) -- Pequeno delay entre upgrades
            end
        end
        if boughtCount > 0 then
            Log(boughtCount .. " upgrades comprados")
        end
    end
end

-- ========== SISTEMA DE INTERFACE (OPCIONAL) ==========
local function CreateControlUI()
    -- Remove UI antiga se existir
    local screenGui = LocalPlayer.PlayerGui:FindFirstChild("AutoFarmUI")
    if screenGui then
        screenGui:Destroy()
    end
    
    -- Cria nova UI
    local ScreenGui = Instance.new("ScreenGui")
    ScreenGui.Name = "AutoFarmUI"
    ScreenGui.Parent = LocalPlayer.PlayerGui
    ScreenGui.ResetOnSpawn = false
    
    local Frame = Instance.new("Frame")
    Frame.Size = UDim2.new(0, 200, 0, 250)
    Frame.Position = UDim2.new(0, 10, 0, 10)
    Frame.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
    Frame.Parent = ScreenGui
    
    local Title = Instance.new("TextLabel")
    Title.Text = "⚙️ AutoFarm Control"
    Title.Size = UDim2.new(1, 0, 0, 30)
    Title.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    Title.TextColor3 = Color3.new(1, 1, 1)
    Title.Parent = Frame
    
    -- Cria toggles para cada funcionalidade
    local yOffset = 35
    local toggles = {}
    
    local function CreateToggle(text, configKey, yPos)
        local toggle = Instance.new("TextButton")
        toggle.Size = UDim2.new(0.9, 0, 0, 25)
        toggle.Position = UDim2.new(0.05, 0, 0, yPos)
        toggle.Text = text .. ": " .. (CONFIG[configKey] and "✅" or "❌")
        toggle.BackgroundColor3 = CONFIG[configKey] and Color3.fromRGB(0, 100, 0) or Color3.fromRGB(100, 0, 0)
        toggle.TextColor3 = Color3.new(1, 1, 1)
        
        toggle.MouseButton1Click:Connect(function()
            CONFIG[configKey] = not CONFIG[configKey]
            toggle.Text = text .. ": " .. (CONFIG[configKey] and "✅" or "❌")
            toggle.BackgroundColor3 = CONFIG[configKey] and Color3.fromRGB(0, 100, 0) or Color3.fromRGB(100, 0, 0)
            Log(text .. " alterado para: " .. tostring(CONFIG[configKey]))
        end)
        
        toggle.Parent = Frame
        toggles[configKey] = toggle
        return yPos + 30
    end
    
    yOffset = CreateToggle("Auto Roll", "EnableAutoRoll", yOffset)
    yOffset = CreateToggle("Auto Sell", "EnableAutoSell", yOffset)
    yOffset = CreateToggle("Auto Upgrade", "EnableAutoUpgrade", yOffset)
    yOffset = CreateToggle("Auto Floor", "EnableAutoFloor", yOffset)
    yOffset = CreateToggle("Auto Equip", "EnableAutoEquip", yOffset)
    
    -- Botão para fechar UI
    local CloseButton = Instance.new("TextButton")
    CloseButton.Text = "Fechar"
    CloseButton.Size = UDim2.new(0.4, 0, 0, 25)
    CloseButton.Position = UDim2.new(0.3, 0, 0, yOffset)
    CloseButton.BackgroundColor3 = Color3.fromRGB(100, 100, 100)
    CloseButton.TextColor3 = Color3.new(1, 1, 1)
    CloseButton.Parent = Frame
    
    CloseButton.MouseButton1Click:Connect(function()
        ScreenGui.Enabled = false
    end)
    
    -- Botão para reabrir UI (minimizado)
    local OpenButton = Instance.new("TextButton")
    OpenButton.Text = "⚙️"
    OpenButton.Size = UDim2.new(0, 30, 0, 30)
    OpenButton.Position = UDim2.new(0, 10, 0, 10)
    OpenButton.BackgroundColor3 = Color3.fromRGB(60, 60, 60)
    OpenButton.TextColor3 = Color3.new(1, 1, 1)
    OpenButton.Visible = false
    OpenButton.Parent = ScreenGui
    
    CloseButton.MouseButton1Click:Connect(function()
        Frame.Visible = false
        OpenButton.Visible = true
    end)
    
    OpenButton.MouseButton1Click:Connect(function()
        Frame.Visible = true
        OpenButton.Visible = false
    end)
end

-- ========== INICIALIZAÇÃO DO SCRIPT ==========
Log("Script de automação iniciado")

-- Cria interface de controle (opcional)
local success = pcall(CreateControlUI)
if success then
    Log("Interface de controle criada")
else
    Log("Interface de controle não pôde ser criada")
end

-- Remove elementos da UI do jogo (se necessário)
local function CleanGameUI()
    pcall(function()
        local CoreGui = game:GetService("CoreGui")
        local PurchasePrompt = CoreGui:FindFirstChild("PurchasePromptApp")
        if PurchasePrompt then
            PurchasePrompt:Destroy()
        end
    end)
    
    pcall(function()
        local NotificationWindow = LocalPlayer.PlayerGui.MainUI:FindFirstChild("NotificationWindow")
        if NotificationWindow then
            NotificationWindow:Destroy()
        end
    end)
end

-- Executa limpeza uma vez
CleanGameUI()

-- ========== LOOP PRINCIPAL ==========
task.spawn(function()
    while true do
        -- Executa todas as ações de automação
        Automation.RollMachines()
        task.wait(0.05)
        
        Automation.SellStorage()
        task.wait(0.05)
        
        Automation.EquipBest()
        task.wait(0.05)
        
        Automation.BuyFloor()
        task.wait(0.05)
        
        Automation.CheckAndBuyUpgrades()
        
        -- Aguarda delay configurado
        task.wait(CONFIG.LoopDelay)
    end
end)

Log("Sistema de automação rodando")
