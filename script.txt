-- ALL-IN-ONE BLACKHOLE (LocalScript)
-- Single-file: UI (mobile-friendly, draggable) + AlignPosition/AlignOrientation control
-- Coloque como LocalScript em StarterPlayerScripts para testar em seu place.

local Players = game:GetService("Players")
local RunService = game:GetService("RunService")
local UIS = game:GetService("UserInputService")
local Workspace = game:GetService("Workspace")

local player = Players.LocalPlayer
local playerGui = player:WaitForChild("PlayerGui")

-- ====== PARAMS (defaults) ======
local params = {
    Enabled = false,
    Radius = 18,            -- distância da forma (studs)
    OrbitSpeed = 40,        -- graus por segundo (velocidade de órbita)
    MagnetRange = 100,      -- raio de captura (studs)
    MaxObjects = 40,        -- máximo (1-200)
    Shape = "Sphere",       -- Sphere | Ring | SphereRing | Cube | Pyramid
    GridSize = 10,          -- para Cube/Pyramid (número de células por lado)
    Offset = Vector3.new(0, 4, 0), -- deslocamento relativo ao HRP
    ShapeRotation = Vector3.new(0, 60, 0), -- deg/s em cada eixo (visual)
}

-- ====== STORAGE ======
local controlled = {}     -- array of parts (keeps ordering)
local controlMap = {}     -- map part -> data {attach, alignPos, alignOri, index}
local orbitAngle = 0      -- radians

-- ====== CENTER PART (invisible) ======
local centerPart = Instance.new("Part")
centerPart.Name = "BH_Center"
centerPart.Size = Vector3.new(0.2,0.2,0.2)
centerPart.Transparency = 1
centerPart.Anchored = true
centerPart.CanCollide = false
centerPart.Parent = Workspace

local centerAttachment = Instance.new("Attachment")
centerAttachment.Name = "BH_CenterAttach"
centerAttachment.Parent = centerPart

-- ====== UTIL ======
local function clamp(v,a,b) if v < a then return a end if v > b then return b end return v end

local function tableRemovePart(part)
    for i,p in ipairs(controlled) do
        if p == part then
            table.remove(controlled, i)
            break
        end
    end
    controlMap[part] = nil
    if part and part.Parent then
        -- try cleanup attachments/aligns we created
        for _,c in ipairs(part:GetChildren()) do
            if c:IsA("Attachment") and c.Name == "BH_Attach" then c:Destroy() end
            if c:IsA("AlignPosition") and c.Name == "BH_AlignPos" then c:Destroy() end
            if c:IsA("AlignOrientation") and c.Name == "BH_AlignOri" then c:Destroy() end
        end
        pcall(function() part.CanCollide = true end)
    end
end

local function clearAll()
    for _,part in ipairs(controlled) do
        controlMap[part] = nil
        if part and part.Parent then
            for _,c in ipairs(part:GetChildren()) do
                if c:IsA("Attachment") and c.Name == "BH_Attach" then c:Destroy() end
                if c:IsA("AlignPosition") and c.Name == "BH_AlignPos" then c:Destroy() end
                if c:IsA("AlignOrientation") and c.Name == "BH_AlignOri" then c:Destroy() end
            end
            pcall(function() part.CanCollide = true end)
        end
    end
    controlled = {}
end

-- ====== VALID PART CHECK ======
local function isValidCandidate(part)
    if not part or not part:IsA("BasePart") then return false end
    if part.Anchored then return false end
    if not part.Parent then return false end
    -- ignore player's character
    if player.Character and part:IsDescendantOf(player.Character) then return false end
    -- ignore models with Humanoid (players/NPCs)
    if part:FindFirstAncestorWhichIsA and part:FindFirstAncestorWhichIsA("Model") then
        local anc = part:FindFirstAncestorWhichIsA("Model")
        if anc and anc:FindFirstChildOfClass("Humanoid") then return false end
    end
    -- ignore very large parts
    if part.Size.Magnitude > 100 then return false end
    return true
end

-- ====== CONTROL (create Attach + Aligns) ======
local function claimPart(part)
    if controlMap[part] then return end
    if #controlled >= clamp(params.MaxObjects,1,200) then return end
    if not isValidCandidate(part) then return end

    -- create small attachment in the part
    local attach = Instance.new("Attachment")
    attach.Name = "BH_Attach"
    attach.Parent = part

    -- create AlignPosition (OneAttachment mode) => Attachment0 set, Attachment1 nil
    local ap = Instance.new("AlignPosition")
    ap.Name = "BH_AlignPos"
    ap.Attachment0 = attach
    ap.Attachment1 = nil
    ap.ReactionForceEnabled = false
    ap.MaxForce = 1e6
    ap.Responsiveness = 200
    ap.RelativeTo = Enum.ActuatorRelativeTo.World
    ap.Position = part.Position
    ap.Parent = part

    -- create AlignOrientation
    local ao = Instance.new("AlignOrientation")
    ao.Name = "BH_AlignOri"
    ao.Attachment0 = attach
    ao.Attachment1 = nil
    ao.ReactionTorqueEnabled = false
    ao.MaxTorque = 1e6
    ao.Responsiveness = 200
    ao.RelativeTo = Enum.ActuatorRelativeTo.World
    ao.Orientation = Vector3.new(0, part.Orientation.Y, 0)
    ao.Parent = part

    part.CanCollide = false
    -- optional: reduce physics jitter
    pcall(function()
        part.CustomPhysicalProperties = PhysicalProperties.new(0.1, 0.3, 0.5, 1, 1)
    end)

    table.insert(controlled, part)
    controlMap[part] = {attach = attach, alignPos = ap, alignOri = ao}
end

-- ====== SHAPE GENERATORS ======
local function fibonacciSphere(n)
    -- returns table of many points on unit sphere (n points)
    local pts = {}
    local gr = (1 + math.sqrt(5)) / 2
    local ga = (2 - gr) * (2 * math.pi)
    for i=1,n do
        local lat = math.asin(-1 + 2 * i / (n + 1))
        local lon = ga * i
        local x = math.cos(lon) * math.cos(lat)
        local y = math.sin(lat)
        local z = math.sin(lon) * math.cos(lat)
        table.insert(pts, Vector3.new(x,y,z))
    end
    return pts
end

local function computeSlotsForCube(grid)
    local slots = {}
    local spacing = 2 * params.Radius / math.max(1, grid - 1)
    for xi=0,grid-1 do
        for yi=0,grid-1 do
            for zi=0,grid-1 do
                local x = (xi - (grid-1)/2) * spacing
                local y = (yi - (grid-1)/2) * spacing
                local z = (zi - (grid-1)/2) * spacing
                table.insert(slots, Vector3.new(x,y,z))
            end
        end
    end
    return slots
end

local function computeSlotsForPyramid(grid)
    local slots = {}
    local spacing = 2 * params.Radius / math.max(1, grid - 1)
    for layer = 0, grid-1 do
        local size = grid - layer
        local y = (-params.Radius) + layer * spacing
        for xi=0,size-1 do
            for zi=0,size-1 do
                local x = (xi - (size-1)/2) * spacing
                local z = (zi - (size-1)/2) * spacing
                table.insert(slots, Vector3.new(x, y, z))
            end
        end
    end
    return slots
end

local function computeTargets(count)
    local targets = {}
    local center = centerPart.Position
    if count <= 0 then return targets end

    if params.Shape == "Ring" then
        for i=1,count do
            local a = (2*math.pi/count) * (i-1)
            local localPos = Vector3.new(math.cos(a), 0, math.sin(a)) * params.Radius
            table.insert(targets, localPos)
        end

    elseif params.Shape == "Sphere" then
        -- generate more points than needed, then sample evenly
        local needed = math.max(count, 200)
        local pts = fibonacciSphere(needed)
        -- pick evenly spaced indices
        local step = math.floor(#pts / count)
        step = math.max(1, step)
        local idx = 1
        for i=1,count do
            local v = pts[idx]
            table.insert(targets, v * params.Radius)
            idx = idx + step
            if idx > #pts then idx = (idx % #pts) + 1 end
        end

    elseif params.Shape == "SphereRing" then
        local half = math.ceil(count/2)
        local ringCount = half
        local sphereCount = count - half
        -- ring
        for i=1,ringCount do
            local a = (2*math.pi/ringCount) * (i-1)
            table.insert(targets, Vector3.new(math.cos(a), 0, math.sin(a)) * params.Radius)
        end
        -- sphere top
        if sphereCount > 0 then
            local pts = fibonacciSphere(math.max(200, sphereCount*3))
            local step = math.floor(#pts / sphereCount)
            step = math.max(1, step)
            local idx = 1
            local added = 0
            while added < sphereCount do
                local v = pts[idx]
                if v.Y > 0 then -- prefer upper hemisphere
                    table.insert(targets, v * params.Radius)
                    added = added + 1
                end
                idx = idx + step
                if idx > #pts then idx = (idx % #pts) + 1 end
            end
        end

    elseif params.Shape == "Cube" then
        local grid = clamp(math.floor(params.GridSize), 1, 100)
        local slots = computeSlotsForCube(grid)
        -- ensure edges visible: sort slots by distance from center descending so edges first
        table.sort(slots, function(a,b) return a.Magnitude > b.Magnitude end)
        -- select evenly among slots
        local step = math.floor(#slots / count)
        step = math.max(1, step)
        local idx = 1
        for i=1,count do
            table.insert(targets, slots[idx])
            idx = idx + step
            if idx > #slots then idx = (idx % #slots) + 1 end
        end

    elseif params.Shape == "Pyramid" then
        local grid = clamp(math.floor(params.GridSize), 1, 100)
        local slots = computeSlotsForPyramid(grid)
        table.sort(slots, function(a,b) return a.Magnitude > b.Magnitude end)
        local step = math.floor(#slots / count)
        step = math.max(1, step)
        local idx = 1
        for i=1,count do
            table.insert(targets, slots[idx])
            idx = idx + step
            if idx > #slots then idx = (idx % #slots) + 1 end
        end
    end

    return targets
end

-- ====== DETECTION LOOP (claim nearby parts) ======
local function scanForParts()
    if #controlled >= clamp(params.MaxObjects,1,200) then return end
    local center = centerPart.Position
    -- use GetPartBoundsInRadius when available (more efficient)
    local found = {}
    pcall(function()
        local parts = Workspace:GetPartBoundsInRadius(center, params.MagnetRange)
        for _,p in ipairs(parts) do
            if #controlled >= clamp(params.MaxObjects,1,200) then break end
            if isValidCandidate(p) and not controlMap[p] then
                claimPart(p)
            end
        end
    end)
    -- fallback brute-force if needed (rare)
    if #controlled < clamp(params.MaxObjects,1,200) then
        for _,p in ipairs(Workspace:GetDescendants()) do
            if #controlled >= clamp(params.MaxObjects,1,200) then break end
            if p:IsA("BasePart") and (p.Position - center).Magnitude <= params.MagnetRange then
                if isValidCandidate(p) and not controlMap[p] then
                    claimPart(p)
                end
            end
        end
    end
end

-- ====== UPDATE LOOP ======
local lastTick = 0
RunService.Heartbeat:Connect(function(dt)
    if not params.Enabled then return end

    -- update center to player position + offset (if player exists)
    if player.Character and player.Character:FindFirstChild("HumanoidRootPart") then
        centerPart.Position = player.Character.HumanoidRootPart.Position + params.Offset
    end

    -- periodic scan (every 0.45s)
    lastTick = lastTick + dt
    if lastTick >= 0.45 then
        lastTick = 0
        scanForParts()
    end

    local count = #controlled
    if count == 0 then return end

    orbitAngle = orbitAngle + math.rad(params.OrbitSpeed) * dt

    -- compute targets
    local targets = computeTargets(count)

    -- shape rotation (convert deg/s to rad increment)
    local rotX = math.rad(params.ShapeRotation.X) * dt
    local rotY = math.rad(params.ShapeRotation.Y) * dt
    local rotZ = math.rad(params.ShapeRotation.Z) * dt
    -- accumulate CFrame rotation per frame via orbitAngle (we rotate by orbitAngle around Y)
    for i,part in ipairs(controlled) do
        if part and part.Parent and controlMap[part] then
            local data = controlMap[part]
            local localPos = targets[i] or Vector3.new(0,0,0)
            -- rotate localPos by the orbitAngle on Y plus incremental shape rotation
            local rotC = CFrame.Angles(rotX * i, orbitAngle, rotZ * i) -- slight per-index variation to look nice
            local worldTarget = (CFrame.new(centerPart.Position) * rotC * CFrame.new(localPos)).p

            -- set AlignPosition.Position (OneAttachment, RelativeTo = World)
            pcall(function()
                data.alignPos.Position = worldTarget
            end)

            -- orient to face center (so objects look inward)
            local dir = (centerPart.Position - part.Position)
            if dir.Magnitude > 0.0001 then
                local cf = CFrame.new(Vector3.new(), dir.Unit) -- look direction
                local rx, ry, rz = cf:ToEulerAnglesXYZ() -- radians
                data.alignOri.Orientation = Vector3.new(math.deg(rx), math.deg(ry), math.deg(rz))
            end
        else
            -- cleanup if missing
            if controlMap[part] then tableRemovePart(part) end
        end
    end
end)

-- ====== UI (mobile friendly, draggable) ======
local function createUI()
    -- remove existing
    local old = playerGui:FindFirstChild("BH_UI")
    if old then old:Destroy() end

    local screenGui = Instance.new("ScreenGui")
    screenGui.Name = "BH_UI"
    screenGui.ResetOnSpawn = false
    screenGui.IgnoreGuiInset = true
    screenGui.Parent = playerGui

    local main = Instance.new("Frame", screenGui)
    main.Name = "Main"
    main.AnchorPoint = Vector2.new(0.5,0.5)
    main.Position = UDim2.fromScale(0.5, 0.5)
    main.Size = UDim2.fromScale(0.36, 0.58) -- mobile-friendly
    main.BackgroundColor3 = Color3.fromRGB(24,24,24)
    main.BorderSizePixel = 0
    main.ClipsDescendants = true

    local uilist = Instance.new("UIListLayout", main)
    uilist.Padding = UDim.new(0,8)
    uilist.SortOrder = Enum.SortOrder.LayoutOrder
    uilist.FillDirection = Enum.FillDirection.Vertical

    -- Title
    local title = Instance.new("TextLabel", main)
    title.Size = UDim2.new(1, -16, 0, 36)
    title.Position = UDim2.new(0,8,0,8)
    title.BackgroundTransparency = 1
    title.Text = "BlackHole Controller"
    title.TextColor3 = Color3.fromRGB(220,220,220)
    title.Font = Enum.Font.GothamBold
    title.TextSize = 18
    title.LayoutOrder = 0

    -- small helper to create a labeled slider (supports decimal via stepArg)
    local function addSlider(labelText, min, max, initial, step, onChange)
        local container = Instance.new("Frame", main)
        container.Size = UDim2.new(1, -12, 0, 56)
        container.BackgroundTransparency = 1
        container.LayoutOrder = 1

        local lbl = Instance.new("TextLabel", container)
        lbl.Size = UDim2.new(0.5, 0, 0, 20)
        lbl.Position = UDim2.new(0,6,0,6)
        lbl.BackgroundTransparency = 1
        lbl.Text = labelText .. " : " .. tostring(initial)
        lbl.TextColor3 = Color3.fromRGB(230,230,230)
        lbl.Font = Enum.Font.Gotham
        lbl.TextSize = 14

        local bg = Instance.new("Frame", container)
        bg.Position = UDim2.new(0,6,0,28)
        bg.Size = UDim2.new(1, -12, 0, 16)
        bg.BackgroundColor3 = Color3.fromRGB(60,60,60)
        bg.ClipsDescendants = true

        local fill = Instance.new("Frame", bg)
        local frac = (initial - min) / math.max(1, (max - min))
        fill.Size = UDim2.new(frac, 0, 1, 0)
        fill.BackgroundColor3 = Color3.fromRGB(0,170,255)

        local dragging = false
        local function setFromX(x)
            local pos = clamp((x - bg.AbsolutePosition.X) / bg.AbsoluteSize.X, 0, 1)
            fill.Size = UDim2.new(pos, 0, 1, 0)
            local raw = min + (max - min) * pos
            local val
            if step and step < 1 then
                val = math.floor((raw / step) + 0.5) * step
                val = math.floor(val*10)/10
            else
                val = math.floor(raw + 0.5)
            end
            lbl.Text = labelText .. " : " .. tostring(val)
            onChange(val)
        end

        bg.InputBegan:Connect(function(input)
            if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
                dragging = true
                setFromX(input.Position.X)
            end
        end)
        bg.InputEnded:Connect(function(input)
            if input.UserInputState == Enum.UserInputState.End then dragging = false end
        end)
        UIS.InputChanged:Connect(function(input)
            if dragging and input.UserInputType == Enum.UserInputType.MouseMovement or (dragging and input.UserInputType == Enum.UserInputType.Touch) then
                setFromX(input.Position.X)
            end
        end)
    end

    -- Dropdown for Shape
    local function addDropdown()
        local container = Instance.new("Frame", main)
        container.Size = UDim2.new(1, -12, 0, 44)
        container.BackgroundTransparency = 1

        local lbl = Instance.new("TextLabel", container)
        lbl.Size = UDim2.new(0.6, 0, 1, 0)
        lbl.BackgroundTransparency = 1
        lbl.Text = "Shape: " .. params.Shape
        lbl.TextColor3 = Color3.fromRGB(230,230,230)
        lbl.Font = Enum.Font.Gotham
        lbl.TextSize = 14

        local btn = Instance.new("TextButton", container)
        btn.Size = UDim2.new(0.35, 0, 0.75, 0)
        btn.Position = UDim2.new(0.62, 0, 0.12, 0)
        btn.Text = "Change"
        btn.BackgroundColor3 = Color3.fromRGB(70,70,70)
        btn.TextColor3 = Color3.fromRGB(240,240,240)
        btn.Font = Enum.Font.GothamBold
        btn.TextSize = 12

        local list = Instance.new("Frame", screenGui)
        list.Size = UDim2.new(0.25, 0, 0.2, 0)
        list.Position = UDim2.new(0.6, 0, 0.4, 0)
        list.BackgroundColor3 = Color3.fromRGB(40,40,40)
        list.Visible = false
        list.ClipsDescendants = true

        local opts = {"Sphere", "Ring", "SphereRing", "Cube", "Pyramid"}
        local y = 6
        for _,opt in ipairs(opts) do
            local b = Instance.new("TextButton", list)
            b.Size = UDim2.new(1, -12, 0, 30)
            b.Position = UDim2.new(0, 6, 0, y)
            b.Text = opt
            b.BackgroundColor3 = Color3.fromRGB(80,80,80)
            b.TextColor3 = Color3.fromRGB(240,240,240)
            b.Font = Enum.Font.Gotham
            b.TextSize = 14
            y = y + 34
            b.MouseButton1Click:Connect(function()
                params.Shape = opt
                lbl.Text = "Shape: " .. opt
                list.Visible = false
            end)
        end

        btn.MouseButton1Click:Connect(function()
            list.Visible = not list.Visible
        end)
    end

    -- Toggle Button + Clear
    local btnToggle = Instance.new("TextButton", main)
    btnToggle.Size = UDim2.new(1, -12, 0, 36)
    btnToggle.Text = "Start"
    btnToggle.Font = Enum.Font.GothamBold
    btnToggle.TextSize = 16
    btnToggle.BackgroundColor3 = Color3.fromRGB(0,150,255)
    btnToggle.TextColor3 = Color3.fromRGB(255,255,255)
    btnToggle.LayoutOrder = 100
    btnToggle.MouseButton1Click:Connect(function()
        params.Enabled = not params.Enabled
        btnToggle.Text = params.Enabled and "Stop" or "Start"
        if not params.Enabled then
            clearAll()
        else
            -- immediate scan when starting
            scanForParts()
        end
    end)

    local btnClear = Instance.new("TextButton", main)
    btnClear.Size = UDim2.new(1, -12, 0, 28)
    btnClear.Text = "Clear Controlled"
    btnClear.Font = Enum.Font.Gotham
    btnClear.TextSize = 14
    btnClear.BackgroundColor3 = Color3.fromRGB(80,80,80)
    btnClear.TextColor3 = Color3.fromRGB(255,255,255)
    btnClear.MouseButton1Click:Connect(function() clearAll() end)

    -- Build UI: sliders + dropdown
    addSlider("Radius", 0, 120, params.Radius, 1, function(v) params.Radius = v end)
    addSlider("OrbitSpeed (°/s)", 0, 360, params.OrbitSpeed, 1, function(v) params.OrbitSpeed = v end)
    addSlider("MagnetRange", 0, 250, params.MagnetRange, 1, function(v) params.MagnetRange = v end)
    addSlider("MaxObjects", 1, 200, params.MaxObjects, 1, function(v) params.MaxObjects = v end)
    addDropdown()
    addSlider("GridSize", 2, 40, params.GridSize, 1, function(v) params.GridSize = v end)
    addSlider("Offset X", -50, 50, params.Offset.X, 1, function(v) params.Offset = Vector3.new(v, params.Offset.Y, params.Offset.Z) end)
    addSlider("Offset Y", -50, 50, params.Offset.Y, 1, function(v) params.Offset = Vector3.new(params.Offset.X, v, params.Offset.Z) end)
    addSlider("Offset Z", -50, 50, params.Offset.Z, 1, function(v) params.Offset = Vector3.new(params.Offset.X, params.Offset.Y, v) end)
    addSlider("ShapeRotation (°/s Y)", 0, 360, params.ShapeRotation.Y, 1, function(v) params.ShapeRotation = Vector3.new(params.ShapeRotation.X, v, params.ShapeRotation.Z) end)

    -- DRAGGABLE: works with Mouse and Touch
    local dragging = false
    local dragStartPos = Vector2.new(0,0)
    local frameStart = main.Position

    main.InputBegan:Connect(function(input)
        if input.UserInputType == Enum.UserInputType.MouseButton1 or input.UserInputType == Enum.UserInputType.Touch then
            dragging = true
            dragStartPos = input.Position
            frameStart = main.Position
            input.Changed:Connect(function()
                if input.UserInputState == Enum.UserInputState.End then dragging = false end
            end)
        end
    end)

    UIS.InputChanged:Connect(function(input)
        if not dragging then return end
        if input.UserInputType == Enum.UserInputType.MouseMovement or input.UserInputType == Enum.UserInputType.Touch then
            local delta = input.Position - dragStartPos
            -- preserve scale, change offsets
            main.Position = UDim2.new(
                frameStart.X.Scale,
                frameStart.X.Offset + delta.X,
                frameStart.Y.Scale,
                frameStart.Y.Offset + delta.Y
            )
        end
    end)
end

-- ====== LISTEN TO DESCENDANT ADDED FOR NEW PARTS (optional) ======
Workspace.DescendantAdded:Connect(function(desc)
    -- if new part appears while enabled and within magnet range, claim
    if not params.Enabled then return end
    if desc:IsA("BasePart") then
        local center = centerPart.Position
        if (desc.Position - center).Magnitude <= params.MagnetRange and isValidCandidate(desc) then
            claimPart(desc)
        end
    end
end)

-- ====== INITIALIZE UI ======
createUI()

-- Script loaded
print("BlackHole All-in-One loaded. UI created (mobile-friendly & draggable).")
