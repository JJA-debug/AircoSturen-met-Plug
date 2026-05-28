# AircoSturen-met-Plug
Project voor het slim aansturen van twee airconditioners met behulp van Shelly Plugs. Bevat de configuratie, scripts en automatiseringen om een aangenamer binnen klimaat te bekomen en energie te besparen.
In de buurt van elke airco heb ik een Powerbaas Bridge gezet. Met deze Powerbaas Bridge kan vrijwel elke merk van airco aangestuurd worden. 
Dit aansturen gaat via een locale API met het versturen van eenvoudige webhooks.
Zoals deze http://192.168.0.151/control?type=10&mode=2&temperature=23&fanspeed=0
Op deze webhook zal de Powerbaas reageren door een infrarood signaal naar de airco te sturen. type=10 = een LG aico, mode=2 = verwarmen, temperature = 23 = 
instellen op 23°C, fanspeed=0 = ventilator snelheid op automatisch.
Deze Webhooks laat ik versturen door een shelly plug die in hetzelfde wifinetwerk zit.
De shelly plug leest ook de 2 Powerbaas Bridges (PB's) uit via een HTTP request. Op die manier komt er via een Jason data uit de 2 PB's.
Deze Jason string bevat ook de temperatuur van de PB's. Op die manier ken ik ook de temperatuur van de ruimte.
Dus de shelly plug kan de PB's sturen en kent ook de omgevingstemperatuur via de temperatuur van de PB's.
De Shelly plug gaat nu elk kwartier beide airco 's aansturen via webhook naar de PB's.De airco's worden zo hoger of lager gestuurd afhankelijk van een 
ingestelde waarde ten opzichte van de werkelijke temperatuur in de ruimte.
In het weekend en s'nachts moet er niet verwarmd of gekoeld worden. Op die momenten worden de airco 's uitgeschakeld door de PB's als de Shelly plug de webhook
stuurt om uit te schakelen.
Ik gebruik de Shelly plug niet om te schakelen, maar gewoon om de Java-script software in te draaien.
Ook heb ik nog een klein webpagina geschreven in de plug zodat de ingestelde temperatuur aan te passen door met een gsm of tabled te browsen op de webpagina 
die gehost wordt door de plug. Ik heb hier niet alleen producten van shelly gebruikt, maar ook van Powerbaas. Dit kan natuurlijk alleen maar omdat Shelly een open
systeem is. Shelly is een open syteem, en dat is volgens mij een bewuste keuze die ik wel weet te warderen.
Op de Webpagina browsen kan met http://192.168.0.XXX/script/1/relay  De 1 gebruiken als het script op plaats 1 staat.

