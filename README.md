local Players = game:GetService("Players")
local UIS = game:GetService("UserInputService")
local RunService = game:GetService("RunService")
local RS = game:GetService("ReplicatedStorage")

local player = Players.LocalPlayer
local pg = player:WaitForChild("PlayerGui")
local isMobile = UIS.TouchEnabled

local GUI_NAME = "RoguePieceSendaiHub"

local old = pg:FindFirstChild(GUI_NAME)
if old then old:Destroy() end

local SENDAI_FOLDER = "Sendai City"
local SENDAI_BOSS_FOLDER = "Boss"
local SUMMON_NPC = "BossSpawnAzula"

local BOSSES = {
    ["Cursed Spirit"] = true,
    ["Yuji"] = true,
    ["Modulo Yuji"] = true,
    ["Sukuna (50%)"] = true,
    ["Gojo (50%)"] = true,
}

local ISLANDS = {
    "Sendai City",
    "Windmill Village",
    "Whispering Jungle",
    "Sandora Island",
    "Glacier Isle",
    "Forge Isle",
    "Jujutsu Highschool",
    "Konoha Village",
    "Hage Island",
    "Huecomundo",
    "Rogue Town",
    "Rogue Town [Backside]",
    "Graveyard of Sword",
    "Graveyard of Sword [Backside]",
    "Throne Isle",
    "Abyss Hill",
    "Abyss Hill [Upper]",
    "Onigashima Town",
    "Soul Society",
    "Innerworld",
    "Dungeon",
    "Tower",
}

local selectedBoss = "Cursed Spirit"
local autoFarm = false
local autoSummon = false
local autoAttack = false
local distance = 25
local currentBoss = nil
local farmThread = 0
local summonThread = 0
local attackThread = 0
local bp, bg, noclipConn
local statusLabel

local function setStatus(t)
    if statusLabel then statusLabel.Text = "● " .. tostring(t) end
end

local function getChar()
    local c = player.Character
    local h = c and c:FindFirstChildOfClass("Humanoid")
    local r = c and c:FindFirstChild("HumanoidRootPart")
    if c and h and h.Health > 0 and r then return c,h,r end
end

local function getRoot()
    local _,_,r = getChar()
    return r
end

local function waitRoot()
    while true do
        local r = getRoot()
        if r then return r end
        task.wait(.2)
    end
end

local function getMobHum(m)
    if not m then return nil end
    return m:FindFirstChildOfClass("Humanoid") or m:FindFirstChildWhichIsA("Humanoid", true)
end

local function getMobRoot(m)
    if not m then return nil end
    if m:IsA("BasePart") then return m end
    return m:FindFirstChild("HumanoidRootPart", true)
        or m:FindFirstChild("UpperTorso", true)
        or m:FindFirstChild("Torso", true)
        or m.PrimaryPart
        or m:FindFirstChildWhichIsA("BasePart", true)
end

local function isAlive(m)
    if not m or not m.Parent then return false end
    local h = getMobHum(m)
    local r = getMobRoot(m)
    return h ~= nil and r ~= nil and h.Health > 0
end

local function clearHold()
    if bp then bp:Destroy(); bp = nil end
    if bg then bg:Destroy(); bg = nil end
end

local function noclip(on)
    if noclipConn then noclipConn:Disconnect(); noclipConn = nil end
    if on then
        noclipConn = RunService.Stepped:Connect(function()
            local c = player.Character
            if c then
                for _,p in ipairs(c:GetDescendants()) do
                    if p:IsA("BasePart") then p.CanCollide = false end
                end
            end
        end)
    end
end

local function smoothTo(cf)
    local root = waitRoot()
    if not bp or bp.Parent ~= root then
        if bp then bp:Destroy() end
        bp = Instance.new("BodyPosition")
        bp.Name = "RPSmoothPosition"
        bp.MaxForce = Vector3.new(1e9,1e9,1e9)
        bp.P = 25000
        bp.D = 1200
        bp.Position = root.Position
        bp.Parent = root
    end
    if not bg or bg.Parent ~= root then
        if bg then bg:Destroy() end
        bg = Instance.new("BodyGyro")
        bg.Name = "RPSmoothGyro"
        bg.MaxTorque = Vector3.new(1e9,1e9,1e9)
        bg.P = 25000
        bg.D = 800
        bg.CFrame = root.CFrame
        bg.Parent = root
    end
    bp.Position = cf.Position
    bg.CFrame = cf
end

local function farmCF(part)
    return CFrame.new(part.Position + Vector3.new(0, tonumber(distance) or 25, 0), part.Position)
end

local function getSendaiBossFolder()
    local s = workspace:FindFirstChild(SENDAI_FOLDER) or workspace:FindFirstChild(SENDAI_FOLDER, true)
    if not s then return nil end
    return s:FindFirstChild(SENDAI_BOSS_FOLDER, true)
end

local function getClosestBoss(name)
    local root = getRoot()
    if not root then return nil end
    local folder = getSendaiBossFolder()
    if not folder then return nil end

    local best, dist = nil, math.huge
    for _,obj in ipairs(folder:GetDescendants()) do
        if obj:IsA("Model") and BOSSES[obj.Name] then
            if (not name or name == "Any" or obj.Name == name) and isAlive(obj) then
                local hrp = getMobRoot(obj)
                if hrp then
                    local d = (hrp.Position - root.Position).Magnitude
                    if d < dist then
                        dist = d
                        best = obj
                    end
                end
            end
        end
    end
    return best
end

local function getSummon()
    return workspace:FindFirstChild(SUMMON_NPC, true)
end

local function tpTo(obj)
    local root = getRoot()
    if not root or not obj then return false end
    local part = getMobRoot(obj)
    if part then
        clearHold()
        root.CFrame = part.CFrame + Vector3.new(0,5,0)
        return true
    end
    return false
end

local function activateSummon()
    local summon = getSummon()
    if not summon then return false, "BossSpawnAzula não encontrado" end

    tpTo(summon)
    task.wait(.25)

    local ok = false
    for _,obj in ipairs(summon:GetDescendants()) do
        if obj:IsA("ProximityPrompt") then
            pcall(function() fireproximityprompt(obj); ok = true end)
        elseif obj:IsA("ClickDetector") then
            pcall(function() fireclickdetector(obj); ok = true end)
        end
    end

    if not ok and summon.Parent then
        for _,obj in ipairs(summon.Parent:GetDescendants()) do
            if obj:IsA("ProximityPrompt") then
                pcall(function() fireproximityprompt(obj); ok = true end)
            elseif obj:IsA("ClickDetector") then
                pcall(function() fireclickdetector(obj); ok = true end)
            end
        end
    end

    return ok, ok and "Summon ativado" or "Prompt do summon não achado"
end

local actionRemote
pcall(function()
    for _,o in ipairs(RS:GetDescendants()) do
        if o:IsA("RemoteEvent") and (o.Name == "ActionRemote" or string.find(o.Name, "Action", 1, true)) then
            actionRemote = o
            break
        end
    end
end)

local function attack()
    if actionRemote then
        pcall(function() actionRemote:FireServer("M1", "Combat") end)
    end
    local c = player.Character
    local tool = c and c:FindFirstChildOfClass("Tool")
    if tool then pcall(function() tool:Activate() end) end
end

local function corner(o,r)
    local c=Instance.new("UICorner"); c.CornerRadius=UDim.new(0,r or 10); c.Parent=o; return c
end
local function stroke(o)
    local s=Instance.new("UIStroke"); s.Color=Color3.fromRGB(90,170,255); s.Transparency=.55; s.Parent=o; return s
end
local function pad(o,l,r,t,b)
    local p=Instance.new("UIPadding")
    p.PaddingLeft=UDim.new(0,l or 0); p.PaddingRight=UDim.new(0,r or l or 0)
    p.PaddingTop=UDim.new(0,t or 0); p.PaddingBottom=UDim.new(0,b or t or 0)
    p.Parent=o; return p
end

local gui=Instance.new("ScreenGui")
gui.Name=GUI_NAME
gui.ResetOnSpawn=false
gui.IgnoreGuiInset=true
gui.DisplayOrder=999999
gui.ZIndexBehavior=Enum.ZIndexBehavior.Sibling
gui.Parent=pg

local main=Instance.new("Frame")
main.Size=isMobile and UDim2.new(.92,0,.72,0) or UDim2.new(0,620,0,430)
main.Position=isMobile and UDim2.new(.04,0,.14,0) or UDim2.new(.5,-310,.5,-215)
main.BackgroundColor3=Color3.fromRGB(8,16,30)
main.BorderSizePixel=0
main.Parent=gui
corner(main,18); stroke(main)

local top=Instance.new("Frame")
top.Size=UDim2.new(1,0,0,50)
top.BackgroundColor3=Color3.fromRGB(12,30,55)
top.BorderSizePixel=0
top.Parent=main

local title=Instance.new("TextLabel")
title.Size=UDim2.new(1,-70,1,0)
title.Position=UDim2.new(0,14,0,0)
title.BackgroundTransparency=1
title.Text="Rogue Piece Hub - Sendai"
title.TextColor3=Color3.fromRGB(180,220,255)
title.Font=Enum.Font.GothamBold
title.TextSize=isMobile and 15 or 19
title.TextXAlignment=Enum.TextXAlignment.Left
title.Parent=top

local close=Instance.new("TextButton")
close.Size=UDim2.new(0,38,0,34)
close.Position=UDim2.new(1,-48,0,8)
close.BackgroundColor3=Color3.fromRGB(220,75,85)
close.Text="×"
close.TextColor3=Color3.new(1,1,1)
close.Font=Enum.Font.GothamBold
close.TextSize=22
close.BorderSizePixel=0
close.Parent=top
corner(close,10)

local side=Instance.new("Frame")
side.Size=isMobile and UDim2.new(0,105,1,-50) or UDim2.new(0,130,1,-50)
side.Position=UDim2.new(0,0,0,50)
side.BackgroundColor3=Color3.fromRGB(10,24,42)
side.BorderSizePixel=0
side.Parent=main
pad(side,8,8,10,10)

local sl=Instance.new("UIListLayout")
sl.Padding=UDim.new(0,8)
sl.SortOrder=Enum.SortOrder.LayoutOrder
sl.Parent=side

local content=Instance.new("Frame")
content.Size=isMobile and UDim2.new(1,-105,1,-84) or UDim2.new(1,-130,1,-84)
content.Position=isMobile and UDim2.new(0,105,0,50) or UDim2.new(0,130,0,50)
content.BackgroundTransparency=1
content.Parent=main
pad(content,10,10,10,10)

local statusBox=Instance.new("Frame")
statusBox.Size=UDim2.new(1,-18,0,28)
statusBox.Position=UDim2.new(0,9,1,-34)
statusBox.BackgroundColor3=Color3.fromRGB(12,30,55)
statusBox.BorderSizePixel=0
statusBox.Parent=main
corner(statusBox,10); stroke(statusBox)

statusLabel=Instance.new("TextLabel")
statusLabel.Size=UDim2.new(1,-12,1,0)
statusLabel.Position=UDim2.new(0,8,0,0)
statusLabel.BackgroundTransparency=1
statusLabel.Text="● Ready"
statusLabel.TextColor3=Color3.fromRGB(120,255,180)
statusLabel.Font=Enum.Font.GothamBold
statusLabel.TextSize=isMobile and 10 or 12
statusLabel.TextXAlignment=Enum.TextXAlignment.Left
statusLabel.Parent=statusBox

local pages={}
local tabs={}

local function button(parent,text,color,h)
    local b=Instance.new("TextButton")
    b.Size=UDim2.new(1,0,0,h or (isMobile and 34 or 38))
    b.BackgroundColor3=color or Color3.fromRGB(45,125,255)
    b.Text=text
    b.TextColor3=Color3.new(1,1,1)
    b.Font=Enum.Font.GothamBold
    b.TextSize=isMobile and 11 or 13
    b.BorderSizePixel=0
    b.Parent=parent
    corner(b,10); stroke(b)
    return b
end

local function label(parent,text)
    local l=Instance.new("TextLabel")
    l.Size=UDim2.new(1,0,0,22)
    l.BackgroundTransparency=1
    l.Text=text
    l.TextColor3=Color3.fromRGB(230,240,255)
    l.Font=Enum.Font.GothamBold
    l.TextSize=isMobile and 11 or 13
    l.TextXAlignment=Enum.TextXAlignment.Left
    l.Parent=parent
    return l
end

local function box(parent,ph,val)
    local x=Instance.new("TextBox")
    x.Size=UDim2.new(1,0,0,isMobile and 34 or 38)
    x.BackgroundColor3=Color3.fromRGB(12,27,48)
    x.Text=val or ""
    x.PlaceholderText=ph or ""
    x.TextColor3=Color3.new(1,1,1)
    x.PlaceholderColor3=Color3.fromRGB(150,170,200)
    x.Font=Enum.Font.GothamBold
    x.TextSize=isMobile and 11 or 13
    x.BorderSizePixel=0
    x.ClearTextOnFocus=false
    x.Parent=parent
    corner(x,10); stroke(x); pad(x,10,10,0,0)
    return x
end

local function card(parent,name)
    local c=Instance.new("Frame")
    c.Size=UDim2.new(1,0,0,80)
    c.BackgroundColor3=Color3.fromRGB(14,32,55)
    c.BorderSizePixel=0
    c.Parent=parent
    corner(c,14); stroke(c); pad(c,12,12,10,12)
    local lay=Instance.new("UIListLayout")
    lay.Padding=UDim.new(0,8)
    lay.SortOrder=Enum.SortOrder.LayoutOrder
    lay.Parent=c
    local t=label(c,name)
    t.TextColor3=Color3.fromRGB(170,220,255)
    lay:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        c.Size=UDim2.new(1,0,0,lay.AbsoluteContentSize.Y+22)
    end)
    return c
end

local function page(name)
    local p=Instance.new("ScrollingFrame")
    p.Name=name
    p.Size=UDim2.new(1,0,1,0)
    p.BackgroundTransparency=1
    p.BorderSizePixel=0
    p.ScrollBarThickness=4
    p.Visible=false
    p.Parent=content
    pad(p,0,5,0,8)
    local l=Instance.new("UIListLayout")
    l.Padding=UDim.new(0,9)
    l.SortOrder=Enum.SortOrder.LayoutOrder
    l.Parent=p
    l:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        p.CanvasSize=UDim2.new(0,0,0,l.AbsoluteContentSize.Y+20)
    end)
    pages[name]=p
    return p
end

local function selectTab(n)
    for k,p in pairs(pages) do p.Visible=(k==n) end
    for k,b in pairs(tabs) do
        b.BackgroundColor3=(k==n) and Color3.fromRGB(45,125,255) or Color3.fromRGB(20,45,75)
    end
    title.Text="Rogue Piece Hub - "..n
end

local function tab(name,icon,order)
    local b=button(side,icon.." "..name,Color3.fromRGB(20,45,75),isMobile and 36 or 40)
    b.LayoutOrder=order
    tabs[name]=b
    local p=page(name)
    b.MouseButton1Click:Connect(function() selectTab(name) end)
    return p
end

local function dropdown(parent,titleText,opts,default,cb)
    local open=false
    local holder=Instance.new("Frame")
    holder.Size=UDim2.new(1,0,0,isMobile and 34 or 38)
    holder.BackgroundTransparency=1
    holder.ClipsDescendants=false
    holder.Parent=parent
    local mainBtn=button(holder,titleText..": "..tostring(default).." ▼")
    mainBtn.Size=UDim2.new(1,0,0,isMobile and 34 or 38)
    mainBtn.ZIndex=40
    local drop=Instance.new("Frame")
    drop.Visible=false
    drop.Size=UDim2.new(1,0,0,0)
    drop.Position=UDim2.new(0,0,0,isMobile and 38 or 42)
    drop.BackgroundColor3=Color3.fromRGB(8,18,32)
    drop.BorderSizePixel=0
    drop.ClipsDescendants=true
    drop.ZIndex=50
    drop.Parent=holder
    corner(drop,10); stroke(drop)
    local scroll=Instance.new("ScrollingFrame")
    scroll.Size=UDim2.new(1,0,1,0)
    scroll.BackgroundTransparency=1
    scroll.BorderSizePixel=0
    scroll.ScrollBarThickness=4
    scroll.ZIndex=51
    scroll.Parent=drop
    pad(scroll,5,5,5,5)
    local lay=Instance.new("UIListLayout")
    lay.Padding=UDim.new(0,4)
    lay.Parent=scroll
    lay:GetPropertyChangedSignal("AbsoluteContentSize"):Connect(function()
        scroll.CanvasSize=UDim2.new(0,0,0,lay.AbsoluteContentSize.Y+10)
    end)
    for _,opt in ipairs(opts) do
        local ob=button(scroll,tostring(opt),Color3.fromRGB(25,45,75),28)
        ob.ZIndex=52
        ob.MouseButton1Click:Connect(function()
            open=false; drop.Visible=false
            holder.Size=UDim2.new(1,0,0,isMobile and 34 or 38)
            drop.Size=UDim2.new(1,0,0,0)
            mainBtn.Text=titleText..": "..tostring(opt).." ▼"
            if cb then cb(opt) end
        end)
    end
    mainBtn.MouseButton1Click:Connect(function()
        open=not open
        if open then
            drop.Visible=true
            local h=math.clamp(#opts*32+12,40,190)
            holder.Size=UDim2.new(1,0,0,(isMobile and 38 or 42)+h)
            drop:TweenSize(UDim2.new(1,0,0,h),Enum.EasingDirection.Out,Enum.EasingStyle.Quad,.12,true)
        else
            holder.Size=UDim2.new(1,0,0,isMobile and 34 or 38)
            drop:TweenSize(UDim2.new(1,0,0,0),Enum.EasingDirection.Out,Enum.EasingStyle.Quad,.12,true)
            task.delay(.12,function() if not open then drop.Visible=false end end)
        end
    end)
    return mainBtn
end

local mainP=tab("Main","🏠",1)
local sendaiP=tab("Sendai","🔥",2)
local bossP=tab("Boss","👑",3)
local tpP=tab("TP","📍",4)
local cfgP=tab("Config","⚙",5)

local c1=card(mainP,"Rogue Piece Sendai Hub")
label(c1,"Boss folder: workspace['Sendai City'].Boss")
label(c1,"Summon NPC: BossSpawnAzula")
label(c1,"Bosses: Cursed Spirit, Yuji, Modulo Yuji, Sukuna/Gojo 50%")

local s1=card(sendaiP,"🎯 Sendai Target")
dropdown(s1,"Boss",{"Any","Cursed Spirit","Yuji","Modulo Yuji","Sukuna (50%)","Gojo (50%)"},selectedBoss,function(v)
    selectedBoss=v
    currentBoss=nil
    setStatus("Target: "..v)
end)

local s2=card(sendaiP,"⚔ Farm / Summon")
label(s2,"Distance / Height")
local distBox=box(s2,"Distance","25")
local farmBtn=button(s2,"Auto Sendai Boss: OFF",Color3.fromRGB(245,160,55))
local summonBtn=button(s2,"Auto Summon: OFF",Color3.fromRGB(120,70,180))
local attackBtn=button(s2,"Auto Attack: OFF",Color3.fromRGB(220,75,85))
local tpSummonBtn=button(s2,"TP BossSpawnAzula",Color3.fromRGB(45,125,255))
label(s2,"Obs: o menu/requisito do summon é controlado pelo próprio jogo.")

local b1=card(bossP,"👑 Manual Boss")
local findBtn=button(b1,"Find Closest Boss",Color3.fromRGB(45,125,255))
local tpBossBtn=button(b1,"TP Boss",Color3.fromRGB(45,125,255))

local t1=card(tpP,"📍 Islands")
dropdown(t1,"Island",ISLANDS,"Sendai City",function(v)
    local obj=workspace:FindFirstChild(v,true)
    if obj and tpTo(obj) then setStatus("TP: "..v) else setStatus("Island not found: "..v) end
end)

local g1=card(cfgP,"⚙ Config")
local noclipBtn=button(g1,"Noclip: ON",Color3.fromRGB(40,170,120))
local resetBtn=button(g1,"Reset Hold",Color3.fromRGB(120,70,180))
label(g1,"Mobile: arraste a barra superior.")

local function setFarm(v)
    autoFarm=v==true
    farmThread+=1
    farmBtn.Text="Auto Sendai Boss: "..(autoFarm and "ON" or "OFF")
    farmBtn.BackgroundColor3=autoFarm and Color3.fromRGB(40,220,140) or Color3.fromRGB(245,160,55)
    if not autoFarm then currentBoss=nil; clearHold(); return end
    local id=farmThread
    task.spawn(function()
        while autoFarm and id==farmThread do
            local ok,err=pcall(function()
                distance=tonumber(distBox.Text) or 25
                if not currentBoss or not isAlive(currentBoss) or (selectedBoss~="Any" and currentBoss.Name~=selectedBoss) then
                    currentBoss=getClosestBoss(selectedBoss)
                end
                if currentBoss and isAlive(currentBoss) then
                    local hrp=getMobRoot(currentBoss)
                    if hrp then
                        smoothTo(farmCF(hrp))
                        if autoAttack then attack() end
                        setStatus("Farming: "..currentBoss.Name)
                    end
                else
                    setStatus("Waiting boss: "..selectedBoss)
                    task.wait(.7)
                end
            end)
            if not ok then currentBoss=nil; setStatus("Farm recovered"); warn(err); task.wait(1) end
            task.wait(.15)
        end
    end)
end

local function setSummon(v)
    autoSummon=v==true
    summonThread+=1
    summonBtn.Text="Auto Summon: "..(autoSummon and "ON" or "OFF")
    summonBtn.BackgroundColor3=autoSummon and Color3.fromRGB(40,220,140) or Color3.fromRGB(120,70,180)
    if not autoSummon then return end
    local id=summonThread
    task.spawn(function()
        while autoSummon and id==summonThread do
            local ok,msg=activateSummon()
            setStatus(msg)
            if not autoFarm then setFarm(true) end
            task.wait(3)
        end
    end)
end

local function setAttack(v)
    autoAttack=v==true
    attackThread+=1
    attackBtn.Text="Auto Attack: "..(autoAttack and "ON" or "OFF")
    attackBtn.BackgroundColor3=autoAttack and Color3.fromRGB(40,220,140) or Color3.fromRGB(220,75,85)
    if not autoAttack then return end
    local id=attackThread
    task.spawn(function()
        while autoAttack and id==attackThread do
            attack()
            task.wait(.15)
        end
    end)
end

farmBtn.MouseButton1Click:Connect(function() setFarm(not autoFarm) end)
summonBtn.MouseButton1Click:Connect(function() setSummon(not autoSummon) end)
attackBtn.MouseButton1Click:Connect(function() setAttack(not autoAttack) end)
tpSummonBtn.MouseButton1Click:Connect(function()
    local s=getSummon()
    if s and tpTo(s) then setStatus("TP BossSpawnAzula") else setStatus("BossSpawnAzula not found") end
end)
findBtn.MouseButton1Click:Connect(function()
    currentBoss=getClosestBoss(selectedBoss)
    if currentBoss then setStatus("Found: "..currentBoss.Name) else setStatus("No boss found") end
end)
tpBossBtn.MouseButton1Click:Connect(function()
    if not currentBoss or not isAlive(currentBoss) then currentBoss=getClosestBoss(selectedBoss) end
    if currentBoss and tpTo(currentBoss) then setStatus("TP boss: "..currentBoss.Name) else setStatus("No boss") end
end)
noclipBtn.MouseButton1Click:Connect(function()
    if noclipConn then
        noclip(false); noclipBtn.Text="Noclip: OFF"; noclipBtn.BackgroundColor3=Color3.fromRGB(120,70,180)
    else
        noclip(true); noclipBtn.Text="Noclip: ON"; noclipBtn.BackgroundColor3=Color3.fromRGB(40,170,120)
    end
end)
resetBtn.MouseButton1Click:Connect(function() clearHold(); setStatus("Hold reset") end)
close.MouseButton1Click:Connect(function() main.Visible=false end)

local toggle=Instance.new("TextButton")
toggle.Size=isMobile and UDim2.new(0,58,0,58) or UDim2.new(0,46,0,46)
toggle.Position=UDim2.new(0,14,.5,-29)
toggle.BackgroundColor3=Color3.fromRGB(45,125,255)
toggle.Text="≡"
toggle.TextColor3=Color3.new(1,1,1)
toggle.Font=Enum.Font.GothamBold
toggle.TextSize=isMobile and 28 or 22
toggle.BorderSizePixel=0
toggle.Parent=gui
corner(toggle,15); stroke(toggle)
toggle.MouseButton1Click:Connect(function()
    main.Visible=not main.Visible
    toggle.Text=main.Visible and "×" or "≡"
end)

local dragging=false
local dragStart,startPos
top.InputBegan:Connect(function(input)
    if input.UserInputType==Enum.UserInputType.MouseButton1 or input.UserInputType==Enum.UserInputType.Touch then
        dragging=true
        dragStart=input.Position
        startPos=main.Position
        input.Changed:Connect(function()
            if input.UserInputState==Enum.UserInputState.End or input.UserInputState==Enum.UserInputState.Cancel then dragging=false end
        end)
    end
end)
UIS.InputChanged:Connect(function(input)
    if dragging and (input.UserInputType==Enum.UserInputType.MouseMovement or input.UserInputType==Enum.UserInputType.Touch) then
        local d=input.Position-dragStart
        main.Position=UDim2.new(startPos.X.Scale,startPos.X.Offset+d.X,startPos.Y.Scale,startPos.Y.Offset+d.Y)
    end
end)

player.CharacterAdded:Connect(function()
    task.wait(1)
    clearHold()
    noclip(true)
end)

noclip(true)
selectTab("Sendai")
setStatus("Rogue Piece Sendai Hub loaded")
print("Rogue Piece Sendai Hub loaded")
