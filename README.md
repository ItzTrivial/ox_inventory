<p align='center'><img src='https://i.imgur.com/JxZpgNM.png'/><br>
<a href='https://streamable.com/bggvpg'>Video showcase</a><br>
https://discord.gg/hmcmv3P7YW</p>

<h2 align='center'>Notice</h2><p align='center'>Hasan is no updating the inventory and has stopped development.<br>
<img src='https://i.imgur.com/IZStQrx.png'/><br><br>
Original Showcase --> https://streamable.com/kpvdj3<br>
Hasan's discord is available at https://discord.gg/6FQhKDXBJ6<br><br>
Further development of this resource is continuing at <a href='https://github.com/thelindat/hsn-inventory'>thelindat/hsn-inventory</a>.<br>All stable updates will be submitted to the main repository as pull requests.</p>
<br><br><br>

<h1 align='center'>Installation Guide</h1><p align='center'>Requires either ESX 1.2, ESX Final, or ExtendedMode<br>OneSync must be enabled</p>
<h3 align='center'> <a href='https://github.com/thelindat/hsn-inventory/wiki'>More guides on the wiki</a> </h3>


## ghmattimysql
* Add this function to wait for sql connection to be established
### ghmattimysql-server.lua
```
exports("ready", function (callback)
  Citizen.CreateThread(function ()
      -- add some more error handling
      while GetResourceState('ghmattimysql') ~= 'started' do
          Citizen.Wait(0)
      end
      -- while not exports['mysql-async']:is_ready() do
      --     Citizen.Wait(0)
      -- end
      callback()
  end)
end)
```

<br>

## Modifying your framework (ESX/EXM) - _Updated for 1.4.4_
* Modifications to money related functions have changed _(1.4.4)_
* Modifications to inventory related functions have changed _(1.4.4)_
* Vanilla files: <a href='https://github.com/esx-framework/es_extended/blob/v1-final/server/classes/player.lua'>ESX Final</a> | <a href='https://github.com/esx-framework/es_extended/blob/1.2.0/server/classes/player.lua'>ESX 1.2</a> | <a href='https://github.com/extendedmode/extendedmode/blob/master/server/classes/player.lua'>EXM</a>
### server/classes/player.lua
Insert the below code (replace the existing functions)
```
	self.setAccountMoney = function(accountName, money)
		money = tonumber(money)
		if money >= 0 then
			local account = self.getAccount(accountName)

			if account then
				local prevMoney = account.money
				local newMoney = ESX.Math.Round(money)
				account.money = newMoney

				self.triggerEvent('esx:setAccountMoney', account)
				if accountName ~= 'bank' then
					local itemCount = exports['hsn-inventory']:getItemCount(self.source, accountName)
					self.removeInventoryItem(accountName, itemCount)
					if money > 0 then
						self.addInventoryItem(accountName, money)
					end
				end
			end
		end
	end

	self.addAccountMoney = function(accountName, money)
		money = tonumber(money)
		if money > 0 then
			local account = self.getAccount(accountName)

			if account then
				local newMoney = account.money + ESX.Math.Round(money)
				account.money = newMoney

				self.triggerEvent('esx:setAccountMoney', account)
				if accountName ~= 'bank' then
					local itemCount = exports['hsn-inventory']:getItemCount(self.source, accountName)
					if itemCount < account.money then
						self.addInventoryItem(accountName, money)
					elseif itemCount > account.money then
						self.removeInventoryItem(accountName, money)
					end
				end
			end
		end
	end

	self.removeAccountMoney = function(accountName, money)
		money = tonumber(money)
		if money > 0 then
			local account = self.getAccount(accountName)

			if account then
				local newMoney = account.money - ESX.Math.Round(money)
				account.money = newMoney

				self.triggerEvent('esx:setAccountMoney', account)
				if accountName ~= 'bank' then
					local itemCount = exports['hsn-inventory']:getItemCount(self.source, accountName)
					if itemCount < account.money then
						self.addInventoryItem(accountName, money)
					elseif itemCount > account.money then
						self.removeInventoryItem(accountName, money)
					end
				end
			end
		end
	end

	self.getInventoryItem = function(name, metadata)
		local item = exports["hsn-inventory"]:getItem(self.source, name, metadata)
		if item then
			return item
		end
		return 
	end

	self.addInventoryItem = function(name, count)
		if name and count > 0 then
			count = ESX.Math.Round(count)
			TriggerEvent("hsn-inventory:server:addItem", self.source, name, count)
		end
	end

	self.removeInventoryItem = function(name, count, metadata)
		if name and count > 0 then
			count = ESX.Math.Round(count)
			exports["hsn-inventory"]:removeItem(self.source, name, count, metadata)
		end
	end

	self.setInventoryItem = function(name, count)
		local item = exports["hsn-inventory"]:getItem(self.source, name)
		if item and count >= 0 then
			count = ESX.Math.Round(count)
			if count > item.count then
				self.addInventoryItem(item.name, count - item.count)
			else
				self.removeInventoryItem(item.name, item.count - count)
			end
		end
	end

	self.getWeight = function()
		return 0 --self.weight
	end

	self.getMaxWeight = function()
		return self.maxWeight
	end

	self.canCarryItem = function(name, count)
		return true
	end

	self.canSwapItem = function(firstItem, firstItemCount, testItem, testItemCount)
		return true
	end

	self.useItem = function(name)
		return exports["hsn-inventory"]:useItem(self.source, name)
	end
```

<br>

### server/main.lua
* Replace the event `esx:useItem` with the following  
```
RegisterNetEvent('esx:useItem')
AddEventHandler('esx:useItem', function(source, itemName)
	local xPlayer = ESX.GetPlayerFromId(source)
	local item = xPlayer.getInventoryItem(itemName)
	if item.count > 0 then
		if item.closeonuse then TriggerClientEvent("hsn-inventory:client:closeInventory", source) end
		ESX.UseItem(source, itemName)
	end
end)
```

* Search for `TriggerEvent('esx:playerLoaded', playerId, xPlayer)`, after which add  
`TriggerEvent('hsn-inventory:setplayerInventory', xPlayer.identifier, xPlayer.inventory)`  

<br>

* Search for `if result[1].inventory and result[1].inventory ~= '' then`  
* Remove or comment out the code below that references inventory and items (should be until it mentions `group`) <a href='https://i.imgur.com/cOAx3SU.png'>EXAMPLE</a>  
* Add the following
```
if result[1].inventory and result[1].inventory ~= '' then
	userData.inventory = json.decode(result[1].inventory)
end
```  

<br>

### server/functions.lua
* Search for `ESX.SavePlayer = function(xPlayer, cb)` and at the top insert
```
local inventory = {}
TriggerEvent('hsn-inventory:getplayerInventory', function(data)
	inventory = data
end, xPlayer.identifier)
```
* Inside the MySQL.Async.execute function, replace  
`[@inventory'] = json.encode(xPlayer.getInventory(true))` with `['@inventory'] = json.encode(inventory)`

<br>

## Setting up items
* As long as you have the above edits in place, you can continue to use ESX.RegiserUsableItem as you have been.  
* Alternatively you are able to add items directly to hsn-inventory in `Config.ItemList`  
* Modify the `hsn-inventory:useItem` event to add effects from using an item.  
* Any items registered with hsn will override the default `esx:useItem` event, so don't worry about overlap.
