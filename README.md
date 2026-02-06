-- Checkers: 👑 Kings + Smooth Tweens + Multi-Eat
local TweenService = game:GetService("TweenService")
local ScreenGui = Instance.new("ScreenGui", game:GetService("CoreGui"))
local MainFrame = Instance.new("Frame", ScreenGui)
local TitleBar = Instance.new("Frame", MainFrame)
local Grid = Instance.new("Frame", MainFrame)
local Status = Instance.new("TextLabel", MainFrame)
local ResetBtn = Instance.new("TextButton", MainFrame)
local CloseBtn = Instance.new("TextButton", TitleBar)

-- Настройки UI
MainFrame.Size = UDim2.new(0, 320, 0, 430)
MainFrame.Position = UDim2.new(0.5, -160, 0.5, -215)
MainFrame.BackgroundColor3 = Color3.fromRGB(25, 25, 25)
MainFrame.BorderSizePixel = 0
MainFrame.Active = true
MainFrame.Draggable = true

TitleBar.Size = UDim2.new(1, 0, 0, 35)
TitleBar.BackgroundColor3 = Color3.fromRGB(40, 40, 40)
CloseBtn.Size = UDim2.new(0, 35, 0, 35); CloseBtn.Position = UDim2.new(1, -35, 0, 0)
CloseBtn.Text = "✕"; CloseBtn.TextColor3 = Color3.new(1,1,1); CloseBtn.BackgroundTransparency = 1
CloseBtn.MouseButton1Click:Connect(function() ScreenGui:Destroy() end)

Status.Size = UDim2.new(1, 0, 0, 30); Status.Position = UDim2.new(0, 0, 0, 40)
Status.Text = "Твой ход"; Status.TextColor3 = Color3.new(1,1,1); Status.BackgroundTransparency = 1; Status.TextSize = 18

Grid.Size = UDim2.new(0, 280, 0, 280); Grid.Position = UDim2.new(0.5, -140, 0, 80)
Grid.BackgroundColor3 = Color3.fromRGB(15, 15, 15)

ResetBtn.Size = UDim2.new(0, 140, 0, 35); ResetBtn.Position = UDim2.new(0.5, -70, 1, -45)
ResetBtn.Text = "СБРОСИТЬ"; ResetBtn.BackgroundColor3 = Color3.fromRGB(0, 120, 215); ResetBtn.TextColor3 = Color3.new(1,1,1)

local board = {}
local selected = nil
local turn = "Player"
local isMoving = false

-- Логика ходов
function getMoves(r, c, onlyJumps)
	local moves = {}
	local p = board[r][c]
	if not p or not p.piece then return moves end
	local dirs = {{1,1},{1,-1},{-1,1},{-1,-1}}

	for _, d in pairs(dirs) do
		if p.type == "Normal" then
			if not onlyJumps then
				local nr, nc = r+d[1], c+d[2]
				local fwd = (p.piece == "Player" and d[1] == -1) or (p.piece == "Bot" and d[1] == 1)
				if fwd and nr>=1 and nr<=8 and nc>=1 and nc<=8 and not board[nr][nc].piece then
					table.insert(moves, {r=nr, c=nc, jump=false})
				end
			end
			local jr, jc = r+d[1]*2, c+d[2]*2
			if jr>=1 and jr<=8 and jc>=1 and jc<=8 then
				local mid = board[r+d[1]][c+d[2]]
				if mid.piece and mid.piece ~= p.piece and not board[jr][jc].piece then
					table.insert(moves, {r=jr, c=jc, jump=true})
				end
			end
		else -- Дамка 👑
			for i = 1, 7 do
				local nr, nc = r+d[1]*i, c+d[2]*i
				if nr<1 or nr>8 or nc<1 or nc>8 then break end
				if not board[nr][nc].piece then
					if not onlyJumps then table.insert(moves, {r=nr, c=nc, jump=false}) end
				else
					if board[nr][nc].piece ~= p.piece then
						local jr, jc = nr+d[1], nc+d[2]
						if jr>=1 and jr<=8 and jc>=1 and jc<=8 and not board[jr][jc].piece then
							table.insert(moves, {r=jr, c=jc, jump=true})
						end
					end
					break
				end
			end
		end
	end
	return moves
end

function render()
	for r=1,8 do for c=1,8 do
		local cell = board[r][c]
		cell.btn.BackgroundColor3 = (r+c)%2 ~= 0 and Color3.fromRGB(45,45,45) or Color3.fromRGB(30,30,30)
		if cell.piece then
			cell.btn.Text = cell.type == "King" and "👑" or "●"
			cell.btn.TextColor3 = cell.piece == "Player" and Color3.fromRGB(0,180,255) or Color3.fromRGB(255,50,50)
		else cell.btn.Text = "" end
	end end
	if selected then
		board[selected.r][selected.c].btn.BackgroundColor3 = Color3.fromRGB(80,80,0)
		for _, m in pairs(getMoves(selected.r, selected.c)) do
			board[m.r][m.c].btn.BackgroundColor3 = m.jump and Color3.fromRGB(150,100,0) or Color3.fromRGB(0,100,0)
		end
	end
end

function performMove(r1, c1, r2, c2)
	if isMoving then return end
	isMoving = true
	local p = board[r1][c1]
	local dist = math.abs(r1-r2)
	local targetPos = UDim2.new((c2-1)*0.125, 0, (r2-1)*0.125, 0)
	
	local tw = TweenService:Create(p.btn, TweenInfo.new(0.3, Enum.EasingStyle.Sine), {Position = targetPos})
	p.btn.ZIndex = 10
	tw:Play()
	
	tw.Completed:Connect(function()
		p.btn.ZIndex = 1
		p.btn.Position = UDim2.new((c1-1)*0.125, 0, (r1-1)*0.125, 0)
		
		if dist >= 2 then
			local dr, dc = (r2-r1)/dist, (c2-c1)/dist
			for i=1, dist-1 do board[r1+dr*i][c1+dc*i].piece = nil end
		end

		board[r2][c2].piece, board[r2][c2].type = p.piece, p.type
		board[r1][c1].piece = nil
		
		-- Становление королем 👑
		if (board[r2][c2].piece == "Player" and r2 == 1) or (board[r2][c2].piece == "Bot" and r2 == 8) then
			board[r2][c2].type = "King"
		end

		local nextJumps = getMoves(r2, c2, true)
		if dist >= 2 and #nextJumps > 0 then
			isMoving = false
			if turn == "Bot" then
				task.wait(0.5); performMove(r2, c2, nextJumps[1].r, nextJumps[1].c)
			else
				selected = {r=r2, c=c2}; Status.Text = "БЕЙ ЕЩЕ!"; render()
			end
		else
			turn = (turn == "Player") and "Bot" or "Player"
			isMoving = false; selected = nil
			Status.Text = (turn == "Bot") and "Бот думает..." or "Твой ход"
			render()
			if turn == "Bot" then task.wait(0.6); botThink() end
		end
	end)
end

function botThink()
	if turn ~= "Bot" or isMoving then return end
	local jumps, normals = {}, {}
	for r=1,8 do for c=1,8 do
		if board[r][c].piece == "Bot" then
			for _, m in pairs(getMoves(r,c)) do
				local data = {r1=r, c1=c, r2=m.r, c2=m.c}
				if m.jump then table.insert(jumps, data) else table.insert(normals, data) end
			end
		end
	end end
	local move = (#jumps > 0) and jumps[math.random(1,#jumps)] or (#normals > 0 and normals[math.random(1,#normals)] or nil)
	if move then performMove(move.r1, move.c1, move.r2, move.c2) else Status.Text = "ПОБЕДА!" end
end

-- Инициализация поля
for r=1,8 do
	board[r] = {}
	for c=1,8 do
		local b = Instance.new("TextButton", Grid)
		b.Size = UDim2.new(0.125, 0, 0.125, 0)
		b.Position = UDim2.new((c-1)*0.125, 0, (r-1)*0.125, 0)
		b.BorderSizePixel = 0; b.TextSize = 24; b.Font = Enum.Font.SourceSansBold
		board[r][c] = {btn = b, piece = nil, type = "Normal"}
		
		b.MouseButton1Click:Connect(function()
			if turn ~= "Player" or isMoving then return end
			if board[r][c].piece == "Player" then
				selected = {r=r, c=c}; render()
			elseif selected then
				for _, m in pairs(getMoves(selected.r, selected.c)) do
					if m.r == r and m.c == c then performMove(selected.r, selected.c, r, c); return end
				end
				selected = nil; render()
			end
		end)
	end
end

function reset()
	isMoving = false
	for r=1,8 do for c=1,8 do
		board[r][c].piece = nil; board[r][c].type = "Normal"
		if (r+c)%2 ~= 0 then
			if r <= 3 then board[r][c].piece = "Bot"
			elseif r >= 6 then board[r][c].piece = "Player" end
		end
	end end
	turn = "Player"; Status.Text = "Твой ход"; render()
end

ResetBtn.MouseButton1Click:Connect(reset)
reset()
