-- LocalScript: StarterPlayerScripts
-- สร้าง GUI ปุ่มแกล้ง แล้วส่งคำขอไปยังเซิร์ฟเวอร์ผ่าน ReplicatedStorage.PrankEvent
-- และรับการสั่ง "blind" เพื่อแสดงเอฟเฟกต์หน้าจอมืด

local ReplicatedStorage = game:GetService("ReplicatedStorage")
local Players = game:GetService("Players")
local LocalPlayer = Players.LocalPlayer
local playerGui = LocalPlayer:WaitForChild("PlayerGui")

local event = ReplicatedStorage:WaitForChild("PrankEvent")

-- ====== สร้าง GUI แบบเรียบง่าย (ถ้าอยากเอาไฟล์ GUI ของตัวเองมาแทนได้) ======
local screenGui = Instance.new("ScreenGui")
screenGui.Name = "PrankGUI"
screenGui.ResetOnSpawn = false
screenGui.Parent = playerGui

local frame = Instance.new("Frame")
frame.Size = UDim2.new(0, 320, 0, 170)
frame.Position = UDim2.new(0, 20, 0, 20)
frame.BackgroundTransparency = 0.2
frame.Parent = screenGui

local title = Instance.new("TextLabel")
title.Size = UDim2.new(1, -10, 0, 28)
title.Position = UDim2.new(0, 5, 0, 5)
title.Text = "ปุ่มแกล้ง (พิมพ์ชื่อเพื่อน)"
title.TextScaled = true
title.BackgroundTransparency = 1
title.Parent = frame

local nameBox = Instance.new("TextBox")
nameBox.Size = UDim2.new(1, -10, 0, 28)
nameBox.Position = UDim2.new(0, 5, 0, 40)
nameBox.PlaceholderText = "พิมพ์ชื่อเพื่อน (Exact name)"
nameBox.Parent = frame

-- ฟังก์ชันสร้างปุ่มง่าย ๆ
local function makeButton(text, y, action, extraParams)
    local btn = Instance.new("TextButton")
    btn.Size = UDim2.new(1, -10, 0, 28)
    btn.Position = UDim2.new(0, 5, 0, y)
    btn.Text = text
    btn.Parent = frame
    btn.MouseButton1Click:Connect(function()
        local targetName = nameBox.Text
        if targetName == "" then
            event:FireServer("invalid") -- ส่งให้เซิร์ฟฟิก (เซิร์ฟจะตอบกลับ)
            return
        end
        -- ส่งคำสั่งไปเซิร์ฟเวอร์
        event:FireServer(action, targetName, extraParams)
    end)
    return btn
end

makeButton("ตัวโตชั่วคราว", 78, "grow")
makeButton("ตัวเล็กชั่วคราว", 108, "shrink")
makeButton("เด้งขึ้นฟ้าสูงๆ", 138, "bounce", {height = 220})
-- ปุ่มดึงมาหา
makeButton("ดึงเพื่อนมาหา", 168, "pull")
-- ปุ่มตาบอดชั่วคราว (client-side effect)
makeButton("ตาบอดชั่วคราว 3 วิ", 198, "blind_client", {duration = 3})

-- ====== สร้างเฟรมดำสำหรับเอฟเฟกต์ "ตาบอดชั่วคราว" ======
local blindGui = Instance.new("ScreenGui")
blindGui.Name = "BlindEffectGui"
blindGui.ResetOnSpawn = false
blindGui.Parent = playerGui

local blindFrame = Instance.new("Frame")
blindFrame.Size = UDim2.new(1,0,1,0)
blindFrame.Position = UDim2.new(0,0,0,0)
blindFrame.BackgroundColor3 = Color3.new(0,0,0)
blindFrame.BackgroundTransparency = 1 -- จะค่อย ๆ ลด transparency ให้มืด
blindFrame.Visible = false
blindFrame.Parent = blindGui

-- เอฟเฟกต์: ฟาดหน้าจอมืด แล้วค่อย ๆ กลับมา
local function playBlindEffect(duration, sourceName)
    blindFrame.Visible = true
    -- ใส่ข้อความเล็ก ๆ แสดงว่าใครเป็นคนแกล้ง (optional)
    local label = Instance.new("TextLabel")
    label.Size = UDim2.new(1,0,0,50)
    label.Position = UDim2.new(0,0,0.5,-25)
    label.BackgroundTransparency = 1
    label.TextScaled = true
    label.Text = (sourceName and ("โดย: "..tostring(sourceName))) or ""
    label.TextColor3 = Color3.new(1,1,1)
    label.Parent = blindFrame

    -- ผสม transition: fade-in 0.2s -> ค้าง -> fade-out 0.5s
    local fadeInTime = 0.15
    local fadeOutTime = 0.4
    local holdTime = math.max(0, (duration or 3) - fadeInTime - fadeOutTime)

    -- ฟังก์ชันเปลี่ยน transparency อย่างต่อเนื่อง (เล็ก ๆ)
    local function tweenTransparency(startT, endT, t)
        local steps = math.max(1, math.floor(t / 0.03))
        for i = 1, steps do
            local alpha = i/steps
            blindFrame.BackgroundTransparency = startT + (endT - startT) * alpha
            wait(t/steps)
        end
    end

    tweenTransparency(1, 0.15, fadeInTime)
    wait(holdTime)
    tweenTransparency(0.15, 1, fadeOutTime)

    -- จบ
    label:Destroy()
    blindFrame.Visible = false
    blindFrame.BackgroundTransparency = 1
end

-- รับข้อความจากเซิร์ฟเวอร์ (error/info/blind)
event.OnClientEvent:Connect(function(action, ...)
    if action == "blind" then
        local duration, sourceName = ...
        -- เล่นเอฟเฟกต์หน้าจอมืด (client-side) เท่านั้น — ปลอดภัย
        spawn(function()
            playBlindEffect(duration, sourceName)
        end)

    elseif action == "error" then
        local msg = ...
        -- แจ้งผู้เล่น (simple)
        warn("Prank error: "..tostring(msg))
        -- อาจจะแสดงข้อความใน GUI เพิ่มได้

    elseif action == "info" then
        local msg = ...
        warn("Prank info: "..tostring(msg))
    end
end)

-- Option: ถ้าอยากปิด GUI ในบางคน (เช่นผู้เล่นไม่อยากถูกแกล้ง) สามารถเพิ่มการตั้งค่าได้
