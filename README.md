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

Hieronder de Software:

// BASISVARIABELEN (WEB + RELAIS)
var temperatuur = 21.5; // Gewenste temperatuur
var mode = 2; // Mode -1..3
let lastMode = mode; // onthoud vorige mode
print("SCRIPT GESTART");

// AIRCO VARIABELEN
let merkcode = 20; // PB1
let merkcode2 = 61; // PB2 (aanpassen indien nodig)
let url1="http:";
let url2="http:";

let modee = 0;
let Soltemp = temperatuur;
let temp = 24; // Airco-setpunt
let fanspeedd = 0;

let inp = { 
  GevTemp: Soltemp, 
  Tempp100: 0, 
  Tempp100_2: 0,
  avgTemp: 0,
  TempKoudAan: 24.5,
  TempWarmAan: 19.5,
  uur: 0, 
  dag: 0, 
  UrlText:"" 
};

let out = { teller: 0 };
let isCalling = false;

// URLS
let urlPB1 = "http://pb-airco-273560406610324.local/";
let urlPB2 = "http://pb-airco-154641335341460.local/";
let urlPB3 = "http://pb-airco-154641335341460.local/api/status";

// INSTELBARE OFFSETS (stuurwaarde)
let stuurOffsetPB1 = 0;
let stuurOffsetPB2 = -4;

// VASTE MEET-OFFSETS
const MEET_OFFSET_PB1 = -4.0;
const MEET_OFFSET_PB2 = -4.0;

// WEB ENDPOINT
HTTPServer.registerEndpoint("relay",function(req,res){
  var html="";
  html+="<html><head><meta charset='UTF-8'><meta name='viewport' content='width=device-width, initial-scale=1'><title>Shelly</title></head>";
  html+="<body style='font-family:Arial;text-align:center;margin-top:40px;'><h1>Shelly</h1>";
  html+="<p>Status relais: <b><span id='status'></span></b></p>";
  html+="<button onclick='schakel(true)'>AAN</button><button onclick='schakel(false)'>UIT</button><hr>";
  html+="<p>Temperatuur:</p><input type='number' step='0.1' id='tempInput' value='"+temperatuur.toFixed(1)+"'>";
  html+="<button onclick='updateTemp()'>Update</button><p><b><span id='tempDisplay'>"+temperatuur.toFixed(1)+"</span> °C</b></p><hr>";
  html+="<p>Mode:</p><select id='modeSelect'>";
  let modeNames={"-1":"Off","0":"Auto","1":"Cool","2":"Heat","3":"Dry"};
  for(let i=-1;i<=3;i++){
    html+="<option value='"+i+"' "+(mode==i?"selected":"")+">Mode "+i+" ("+modeNames[i]+")</option>";
  }
  html+="</select><button onclick='updateMode()'>Update</button><p><b><span id='modeDisplay'>"+mode+"</span></b></p>";
  html+="<script>";
  html+="function updateStatus(){fetch('/rpc/Shelly.GetStatus').then(r=>r.json()).then(d=>{var sw=d['switch:0'];status.innerText=sw.output?'AAN':'UIT';});}";
  html+="function schakel(s){fetch('/rpc/Switch.Set?id=0&on='+s).then(()=>updateStatus());}";
  html+="function updateTemp(){var v=parseFloat(tempInput.value);if(!isNaN(v)){fetch('setTemp',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({val:v})});}}";
  html+="function updateMode(){var v=parseInt(modeSelect.value);fetch('setMode',{method:'POST',headers:{'Content-Type':'application/json'},body:JSON.stringify({val:v})});}";
  html+="setInterval(updateStatus,2000);";
  html+="</script></body></html>";
  res.code=200;
  res.headers=[["Content-Type","text/html"]];
  res.body=html;
  res.send();
});

// TEMP
HTTPServer.registerEndpoint("setTemp",function(req,res){
  if(req.body){
    let d=JSON.parse(req.body);
    let v=parseFloat(d.val);
    if(!isNaN(v)){
      temperatuur=parseFloat(v.toFixed(1));
      Soltemp=temperatuur;
      inp.GevTemp=Soltemp;
    }
  }
  res.code=200;
  res.send();
});

// MODE
HTTPServer.registerEndpoint("setMode",function(req,res){
  if(req.body){
    let d=JSON.parse(req.body);
    let v=parseInt(d.val);
    if(v>=-1 && v<=3){
      lastMode = mode;
      mode = v;

      // ✅ éénmalig bij overgang naar mode = 1
      if(mode === 1 && lastMode !== 1){
        temp = 25;
        print("Mode 1 geactiveerd → temp éénmalig op 25°C gezet");
      }

      print("Nieuwe mode:",mode);
    }
  }
  res.code=200;
  res.send();
});

// LOGGING
Timer.set(5000,true,function(){
  print("==== STATUS ====");
  print("Temp PB1:",inp.Tempp100);
  print("Temp PB2:",inp.Tempp100_2);
  print("Gemiddelde:",inp.avgTemp);
  print("Setpoint:",temp);
  print("Mode:",mode);
  print("URL1:",url1);
  print("URL2:",url2);
  print("SollTemp:",temperatuur);
  print("================");
   print("DagVandeWeek:",inp.dag);
},null);

// HOOFDLOGICA
Timer.set(100000,true,function(){

  out.teller++;
  if(out.teller>6)out.teller=1;
  console.log(out.teller);

  // PB1 uitlezen
  if(out.teller===1 && !isCalling){
    isCalling=true;
    Shelly.call("HTTP.GET",{url:urlPB1},function(resp,err){
      if(err===0){
        let data=JSON.parse(resp.body);
        inp.Tempp100=parseFloat(data.temperature.celsius)+MEET_OFFSET_PB1;
      }
      isCalling=false;
    });
  }

  // PB2 uitlezen
  if(out.teller===2 && !isCalling){
    isCalling=true;
    Shelly.call("HTTP.GET",{url:urlPB3},function(resp,err){
      if(err===0){
        let data=JSON.parse(resp.body);
        inp.Tempp100_2=parseFloat(data.temperature.celsius)+MEET_OFFSET_PB2;
      }
      isCalling=false;
    });
  }

  // regeling + gemiddelde
 if(out.teller == 3) {

    inp.avgTemp = (inp.Tempp100 + inp.Tempp100_2) / 2;

    if(inp.GevTemp > inp.avgTemp){
        temp++;
    } else {
        temp--;
    }

    // Boven grens
    if(temp > 26) {
        mode = 2;
        temp = 20;
    }

    // Onder grens
    if(temp < 19) {
        mode = 1;
        temp = 25;
    }

    // Extra logica (blijft zoals bij jou)
    if (inp.avgTemp > inp.TempKoudAan) {
        mode = 1;
        // temp = 26;
    }

    if (inp.avgTemp < inp.TempWarmAan) {
        mode = 2;
        // temp = 19;
    }
}

  // mode bepalen bij nacht en weekend -1
 if (out.teller === 4) {
  let now = new Date();
  inp.uur = now.getHours();
  inp.dag = now.getDay();

 if (
    inp.uur >= 18 || 
    inp.uur < 5 || 
    inp.dag === 0 ||  // zondag
    inp.dag === 6     // zaterdag
  ) {
    modee = -1;
  } else {
    modee = mode;
  }
}

  // PB1 sturen
  if(out.teller===5 && !isCalling){
    isCalling=true;
    let stuurTemp1 = temp + stuurOffsetPB1;
    url1="http://pb-airco-273560406610324.local/control?type="+merkcode+
         "&mode="+modee+
         "&temperature="+stuurTemp1+
         "&fanspeed="+fanspeedd;
    print("PB1:",url1);
    Shelly.call("HTTP.GET",{url:url1},function(r,e){isCalling=false;});
  }

  // PB2 sturen
  if(out.teller===6 && !isCalling){
    isCalling=true;
    let stuurTemp2 = Math.min(23, Math.max(17, temp + stuurOffsetPB2));
    url2="http://pb-airco-154641335341460.local/control?type="+merkcode2+
         "&mode="+modee+
         "&temperature="+stuurTemp2+
         "&fanspeed="+fanspeedd;
    print("PB2:",url2);
    Shelly.call("HTTP.GET",{url:url2},function(r,e){isCalling=false;});
  }

},null);
