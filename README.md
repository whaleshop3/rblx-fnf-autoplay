-- updated 5/12/21
-- should choke less

-- updated 5/16/21
-- should ignore invisible notes
-- added hit chances and a toggle
-- hit chances are a bit rough but should work good enough
-- 한국어 번역 By Whale

-- only tested on Synapse X
-- 시냅스 X 에서만 테스트 했음
local library = loadstring(game:HttpGet("https://pastebin.com/raw/edJT9EGX"))()

local framework, scrollHandler
while true do
	for _, obj in next, getgc(true) do
		if type(obj) == 'table' and rawget(obj, 'GameUI') then
			framework = obj;
			break
		end	
	end

	for _, module in next, getloadedmodules() do
		if module.Name == 'ScrollHandler' then
			scrollHandler = module;
			break;
		end
	end

	if (type(framework) == 'table') and (typeof(scrollHandler) == 'Instance') then
		break
	end

	wait(1)
end

local runService = game:GetService('RunService')
local userInputService = game:GetService('UserInputService')
local client = game:GetService('Players').LocalPlayer;
local random = Random.new()

local fastWait, fastSpawn, fireSignal, rollChance do
	-- https://eryn.io/gist/3db84579866c099cdd5bb2ff37947cec
	-- bla bla spawn and wait are bad 
	-- can also use bindables for the fastspawn idc

    function fastWait(t)
        local d = 0;
        while d < t do
            d += runService.RenderStepped:wait()
        end
    end

    function fastSpawn(f)
        coroutine.wrap(f)()
    end

	function fireSignal(target, signal, ...)
		-- getconnections with InputBegan / InputEnded does not work without setting Synapse to the game's context level
		syn.set_thread_identity(2) 
		for _, signal in next, getconnections(signal) do
			if type(signal.Function) == 'function' and islclosure(signal.Function) then
				local scr = rawget(getfenv(signal.Function), 'script')
				if scr == target then
					pcall(signal.Function, ...)
				end
			end
		end
		syn.set_thread_identity(7)
	end

	-- uses a weighted random system
	-- its a bit scuffed rn but it works good enough

	function rollChance()
		local chances = {
			{ type = 'Sick', value = library.flags.sickChance },
			{ type = 'Good', value = library.flags.goodChance },
			{ type = 'Ok', value = library.flags.okChance },
			{ type = 'Bad', value = library.flags.badChance },
		}
		
		table.sort(chances, function(a, b) 
			return a.value > b.value 
		end)

		local sum = 0;
		for i = 1, #chances do
			sum += chances[i].value
		end

		if sum == 0 then
			return choices[random:NextInteger(1, #choices)]
		end

		local initialWeight = random:NextInteger(0, sum)
		local weight = 0;

		for i = 1, #chances do
			weight = weight + chances[i].value

			if weight > initialWeight then
				return chances[i].type
			end
		end

		return 'Sick' -- just incase it fails?
	end
end

local map = { [0] = 'Left', [1] = 'Down', [2] = 'Up', [3] = 'Right', }
local keys = { Up = Enum.KeyCode.W; Down = Enum.KeyCode.S; Left = Enum.KeyCode.A; Right = Enum.KeyCode.D; }

-- they are "weird" because they are in the middle of their Upper & Lower ranges 
-- should hopefully make them more precise!
local chanceValues = {
	Sick = 96,
	Good = 92,
	Ok = 87,
	Bad = 77,
}

local marked = {}
local hitChances = {}

if shared._id then
	pcall(runService.UnbindFromRenderStep, runService, shared._id)
end

shared._id = game:GetService('HttpService'):GenerateGUID(false)
runService:BindToRenderStep(shared._id, 1, function()
	if (not library.flags.autoPlayer) then return end

	for i, arrow in next, framework.UI.ActiveSections do
		if (arrow.Side == framework.UI.CurrentSide) and (not marked[arrow]) then 
			local indice = (arrow.Data.Position % 4) -- mod 4 because 5%4 -> 0, 6%4 = 1, etc
			local position = map[indice]
			
			if (position) then
				local currentTime = framework.SongPlayer.CurrentlyPlaying.TimePosition
				local distance = (1 - math.abs(arrow.Data.Time - currentTime)) * 100

				if (arrow.Data.Time == 0) then
				--	print('invisible', tableToString(arrow.Data), i, distance)
					continue
				end

				local hitChance = hitChances[arrow] or rollChance()
				hitChances[arrow] = hitChance

				-- if (not chanceValues[hitChance]) then warn('invalid chance', hitChance) end
				if distance >= chanceValues[hitChance] then
					marked[arrow] = true;
					fireSignal(scrollHandler, userInputService.InputBegan, { KeyCode = keys[position], UserInputType = Enum.UserInputType.Keyboard }, false)

					-- wait depending on the arrows length so the animation can play
					if arrow.Data.Length > 0 then
						fastWait(arrow.Data.Length)
					else
						fastWait(0.075) -- 0.1 seems to make it miss more, this should be fine enough?
					end

					fireSignal(scrollHandler, userInputService.InputEnded, { KeyCode = keys[position], UserInputType = Enum.UserInputType.Keyboard }, false)
					marked[arrow] = false;
				end
			end
		end
	end
end)

local window = library:CreateWindow('펑키 프라이데이') do
	local folder = window:AddFolder('메인(오토플레이,자동플레이)') do
		folder:AddToggle({ text = '오토 플레이', flag = 'autoPlayer' })

		folder:AddSlider({ text = 'Sick %', flag = 'sickChance', min = 0, max = 100, value = 100 })
		folder:AddSlider({ text = 'Good %', flag = 'goodChance', min = 0, max = 100, value = 0 })
		folder:AddSlider({ text = 'Ok %', flag = 'okChance', min = 0, max = 100, value = 0 })
		folder:AddSlider({ text = 'Bad %', flag = 'badChance', min = 0, max = 100, value = 0 })
	end

	local folder = window:AddFolder('크레딧') do
		folder:AddLabel({ text = '크레딧' })
		folder:AddLabel({ text = 'Jan - UI 폴더' })
		folder:AddLabel({ text = 'wally - 스크립트' })
        folder:AddLabel({ text = '웨일(Whale) - 한국어 번역' })
	end
end


library:Init()
