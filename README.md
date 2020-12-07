# persistent_vehiclefix_attempt
An attempt at fixing vehicle disappearing on newer version of artifacts (in some cases)
You might get some errors on the server where it tries to access an invalid value; can't solve that problem cba

1. Open '\es_extended\server\main.lua' and add this at the bottom
`
ESX.RegisterServerCallback('sandy:vehiclepersistent', function(source, cb, model, coords, heading)
	local timeout = 60
	local vehicle = CreateVehicle(model, coords.x, coords.y, coords.z, heading, true, true)
    while not DoesEntityExist(vehicle) and timeout > 0 do
    	Citizen.Wait(100)
        timeout = timeout - 1
    end
    timeout = 60
    local netId = NetworkGetNetworkIdFromEntity(vehicle)
	while netId == nil and timeout > 0 do
   		Citizen.Wait(100)
    	netId = NetworkGetNetworkIdFromEntity(vehicle)
    	timeout = timeout - 1
	end
	cb(netId)
end)
`
2. Open '\es_extended\client\client.lua' and edit your 'ESX.Game.SpawnVehicle' function to look like this
`
ESX.Game.SpawnVehicle = function(modelName, coords, heading, cb)
	local model = (type(modelName) == 'number' and modelName or GetHashKey(modelName))

	Citizen.CreateThread(function()
		ESX.Streaming.RequestModel(model)

		if heading == nil then
			heading = coords.w
		end

		local veh = nil

		ESX.TriggerServerCallback('sandy:vehiclepersistent', function(id)
			veh = NetToVeh(id)

			while not DoesEntityExist(veh) do
				veh = NetToVeh(id)
				Citizen.Wait(100)
			end
			
			SetEntityAsMissionEntity(veh, true, false)
			SetVehicleHasBeenOwnedByPlayer(veh, true)
			SetVehicleNeedsToBeHotwired(veh, false)
			SetModelAsNoLongerNeeded(model)

			RequestCollisionAtCoord(coords.x, coords.y, coords.z)

			while not HasCollisionLoadedAroundEntity(veh) do
				RequestCollisionAtCoord(coords.x, coords.y, coords.z)
				Citizen.Wait(0)
			end

			SetVehRadioStation(veh, 'OFF')
			if cb then
				cb(veh)
			end
		end, model, coords, heading)
	end)
end
`
