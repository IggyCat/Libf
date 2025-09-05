-- safe Instance.new wrapper for sUNC / loadstring
local function safeInstance(className, parent)
    local obj
    task.synchronize() -- forces execution in UI-capable thread
    obj = Instance.new(className)
    if parent then obj.Parent = parent end
    return obj
end

local drawingUI = nil
task.spawn(function()
    repeat task.wait() until game and game:GetService("CoreGui")

    drawingUI = safeInstance("ScreenGui", game:GetService("CoreGui"))
    drawingUI.Name = "Drawing"
    drawingUI.IgnoreGuiInset = true
    drawingUI.DisplayOrder = 0x7fffffff
end)

local drawingIndex = 0

local baseDrawingObj = setmetatable({
    Visible = true,
    ZIndex = 0,
    Transparency = 1,
    Color = Color3.new(),
    Remove = function(self)
        setmetatable(self, nil)
    end,
    Destroy = function(self)
        setmetatable(self, nil)
    end
}, {
    __add = function(t1, t2)
        local result = table.clone(t1)
        for index, value in t2 do
            result[index] = value
        end
        return result
    end
})

local drawingFontsEnum = {
    [0] = Font.fromEnum(Enum.Font.Roboto),
    [1] = Font.fromEnum(Enum.Font.Legacy),
    [2] = Font.fromEnum(Enum.Font.SourceSans),
    [3] = Font.fromEnum(Enum.Font.RobotoMono),
}

local function convertTransparency(transparency: number): number
    return math.clamp(1 - transparency, 0, 1)
end

local DrawingLib = {}
DrawingLib.Fonts = {
    ["UI"] = 0,
    ["System"] = 1,
    ["Plex"] = 2,
    ["Monospace"] = 3
}

function DrawingLib.new(drawingType)
    drawingIndex += 1
    if drawingType == "Line" then
        local lineObj = ({
            From = Vector2.zero,
            To = Vector2.zero,
            Thickness = 1
        } + baseDrawingObj)

        local lineFrame = safeInstance("Frame", drawingUI)
        lineFrame.Name = drawingIndex
        lineFrame.AnchorPoint = (Vector2.one * .5)
        lineFrame.BorderSizePixel = 0

        lineFrame.BackgroundColor3 = lineObj.Color
        lineFrame.Visible = lineObj.Visible
        lineFrame.ZIndex = lineObj.ZIndex
        lineFrame.BackgroundTransparency = convertTransparency(lineObj.Transparency)
        lineFrame.Size = UDim2.new()

        return setmetatable({__type = "Drawing Object"}, {
            __newindex = function(_, index, value)
                if typeof(lineObj[index]) == "nil" then return end
                if index == "From" then
                    local direction = (lineObj.To - value)
                    local center = (lineObj.To + value) / 2
                    local distance = direction.Magnitude
                    local theta = math.deg(math.atan2(direction.Y, direction.X))
                    lineFrame.Position = UDim2.fromOffset(center.X, center.Y)
                    lineFrame.Rotation = theta
                    lineFrame.Size = UDim2.fromOffset(distance, lineObj.Thickness)
                elseif index == "To" then
                    local direction = (value - lineObj.From)
                    local center = (value + lineObj.From) / 2
                    local distance = direction.Magnitude
                    local theta = math.deg(math.atan2(direction.Y, direction.X))
                    lineFrame.Position = UDim2.fromOffset(center.X, center.Y)
                    lineFrame.Rotation = theta
                    lineFrame.Size = UDim2.fromOffset(distance, lineObj.Thickness)
                elseif index == "Thickness" then
                    local distance = (lineObj.To - lineObj.From).Magnitude
                    lineFrame.Size = UDim2.fromOffset(distance, value)
                elseif index == "Visible" then
                    lineFrame.Visible = value
                elseif index == "ZIndex" then
                    lineFrame.ZIndex = value
                elseif index == "Transparency" then
                    lineFrame.BackgroundTransparency = convertTransparency(value)
                elseif index == "Color" then
                    lineFrame.BackgroundColor3 = value
                end
                lineObj[index] = value
            end,
            __index = function(self, index)
                if index == "Remove" or index == "Destroy" then
                    return function()
                        lineFrame:Destroy()
                        lineObj.Remove(self)
                        return lineObj:Remove()
                    end
                end
                return lineObj[index]
            end,
            __tostring = function() return "Drawing" end
        })
    elseif drawingType == "Text" then
        local textObj = ({
            Text = "",
            Font = DrawingLib.Fonts.UI,
            Size = 0,
            Position = Vector2.zero,
            Center = false,
            Outline = false,
            OutlineColor = Color3.new()
        } + baseDrawingObj)

        local textLabel = safeInstance("TextLabel")
        local uiStroke = safeInstance("UIStroke", textLabel)
        textLabel.Name = drawingIndex
        textLabel.AnchorPoint = (Vector2.one * .5)
        textLabel.BorderSizePixel = 0
        textLabel.BackgroundTransparency = 1

        textLabel.Visible = textObj.Visible
        textLabel.TextColor3 = textObj.Color
        textLabel.TextTransparency = convertTransparency(textObj.Transparency)
        textLabel.ZIndex = textObj.ZIndex
        textLabel.FontFace = drawingFontsEnum[textObj.Font]
        textLabel.TextSize = textObj.Size

        textLabel:GetPropertyChangedSignal("TextBounds"):Connect(function()
            local textBounds = textLabel.TextBounds
            local offset = textBounds / 2
            textLabel.Size = UDim2.fromOffset(textBounds.X, textBounds.Y)
            textLabel.Position = UDim2.fromOffset(textObj.Position.X + (if not textObj.Center then offset.X else 0), textObj.Position.Y + offset.Y)
        end)

        uiStroke.Thickness = 1
        uiStroke.Enabled = textObj.Outline
        uiStroke.Color = textObj.Color

        textLabel.Parent = drawingUI
        return setmetatable({__type = "Drawing Object"}, {
            __newindex = function(_, index, value)
                if typeof(textObj[index]) == "nil" then return end
                if index == "Text" then
                    textLabel.Text = value
                elseif index == "Font" then
                    value = math.clamp(value, 0, 3)
                    textLabel.FontFace = drawingFontsEnum[value]
                elseif index == "Size" then
                    textLabel.TextSize = value
                elseif index == "Position" then
                    local offset = textLabel.TextBounds / 2
                    textLabel.Position = UDim2.fromOffset(value.X + (if not textObj.Center then offset.X else 0), value.Y + offset.Y)
                elseif index == "Center" then
                    local position = (if value then workspace.CurrentCamera.ViewportSize / 2 else textObj.Position)
                    textLabel.Position = UDim2.fromOffset(position.X, position.Y)
                elseif index == "Outline" then
                    uiStroke.Enabled = value
                elseif index == "OutlineColor" then
                    uiStroke.Color = value
                elseif index == "Visible" then
                    textLabel.Visible = value
                elseif index == "ZIndex" then
                    textLabel.ZIndex = value
                elseif index == "Transparency" then
                    local transparency = convertTransparency(value)
                    textLabel.TextTransparency = transparency
                    uiStroke.Transparency = transparency
                elseif index == "Color" then
                    textLabel.TextColor3 = value
                end
                textObj[index] = value
            end,
            __index = function(self, index)
                if index == "Remove" or index == "Destroy" then
                    return function()
                        textLabel:Destroy()
                        textObj.Remove(self)
                        return textObj:Remove()
                    end
                elseif index == "TextBounds" then
                    return textLabel.TextBounds
                end
                return textObj[index]
            end,
            __tostring = function() return "Drawing" end
        })
    end
    -- repeat the same safeInstance(...) replacement for Circle, Square, Image, Quad, Triangle
end

getgenv().Drawing = DrawingLib

getgenv().isrenderobj = function(obj)
    local s, r = pcall(function()
        return obj.__type == "Drawing Object"
    end)
    return s and r
end
getgenv().cleardrawcache = function()
    if drawingUI then drawingUI:ClearAllChildren() end
end
getgenv().getrenderproperty = function(obj, prop)
    assert(getgenv().isrenderobj(obj), "Object must be a Drawing", 3)
    return obj[prop]
end
getgenv().setrenderproperty = function(obj, prop, val)
    assert(getgenv().isrenderobj(obj), "Object must be a Drawing", 3)
    obj[prop] = val
end
