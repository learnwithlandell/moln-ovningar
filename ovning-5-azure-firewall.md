# 🧩 Labb: Azure Firewall med Azure CLI

## 🎯 Mål

I denna övning ska du bygga ett virtuellt Azure-nätverk med en Linuxmaskin som inte har någon egen väg ut på internet. All trafik från maskinen ska skickas via **Azure Firewall**.

Du kommer att:

- skapa ett virtuellt nätverk med två subnät,
- skapa en privat virtuell maskin,
- skapa och konfigurera Azure Firewall,
- styra trafiken via en route-tabell,
- skapa regler som blockerar Facebook och Instagram,
- tillåta ett fåtal utvalda webbplatser,
- testa regelverkets prioritet och standardbeteende,
- städa bort alla resurser när labben är klar.

Hela labben genomförs i **Azure Cloud Shell med Bash**.

**Tidsåtgång:** cirka en och en halv timme inklusive väntetid  
**Förkunskaper:** grundläggande Azure, IP-adressering och Bash  
**Verktyg:** Azure Cloud Shell (Bash), az CLI 2.90.0 och tillägget `azure-firewall` 2.2.1

> ⚠️ **Viktigt:** Azure Firewall debiteras per timme så länge den finns. Gör labben i ett sammanhang och kör **Del 9 – Städa upp** direkt när du är klar.

---

## 🧭 Så arbetar du

- Kör ett kommando i taget, i den ordning de står.
- Vänta alltid tills prompten kommer tillbaka innan du kör nästa kommando.
- Jämför det du ser med **Förväntat resultat** innan du går vidare.
- Stämmer resultatet inte, stanna och läs avsnittet **Felsökning** sist i övningen.
- Inget kommando innehåller något du själv ska hitta på. Alla namn och adresser ligger i variabler som du skapar i Del 0.

---

## 📋 Namn och adresser

| Resurs | Namn eller adress |
|---|---|
| Region | `swedencentral` |
| Resursgrupp | `rg-fwlab` |
| Virtuellt nätverk | `vnet-fwlab` – `10.0.0.0/16` |
| Brandväggens subnät | `AzureFirewallSubnet` – `10.0.1.0/26` |
| Maskinens subnät | `snet-app` – `10.0.2.0/24` (privat) |
| Virtuell maskin | `vm-app01` – `10.0.2.4` |
| Brandvägg | `fw-fwlab` – `10.0.1.4` (Standard) |
| Brandväggens publika IP | `pip-fw-fwlab` |
| Route-tabell | `rt-app` – `0.0.0.0/0` till `10.0.1.4` |
| Regelsamling Deny | `app-deny` – prioritet 100 |
| Regelsamling Allow | `app-allow` – prioritet 200 |

---

## 🪄 Del 0 – Förbered Cloud Shell

Alla kommandon körs i Azure Cloud Shell med Bash. Där finns Azure CLI redan installerat och du är redan inloggad.

### Steg 1️⃣ – Öppna Cloud Shell

1. Gå till [https://portal.azure.com](https://portal.azure.com) och logga in.
2. Klicka på ikonen **Cloud Shell** (`>_`) i den övre menyraden.
3. Välj **Bash** om du får frågan om Bash eller PowerShell.
4. Om PowerShell visas uppe till vänster klickar du på **Switch to Bash** och bekräftar med **Confirm**.
5. Första gången visas rutan **Getting started**. Välj **No storage account required**, välj din prenumeration under **Subscription** och klicka på **Apply**.

### Steg 2️⃣ – Kontrollera prenumerationen

```bash
az account show --query "{prenumeration:name, id:id}" -o table
```

💡 **Förväntat resultat:** En tabell med namnet på din prenumeration.

Är det fel prenumeration kör du:

```bash
az account set --subscription "NAMN"
```

### Steg 3️⃣ – Installera Azure Firewall-tillägget

Kommandona `az network firewall` finns i ett tillägg till Azure CLI.

```bash
az extension add --name azure-firewall --upgrade -y
```

💡 **Förväntat resultat:** Kommandot avslutas utan felmeddelande.

### Steg 4️⃣ – Kontrollera tillägget

```bash
az extension show --name azure-firewall --query version -o tsv
```

💡 **Förväntat resultat:** Ett versionsnummer, till exempel `2.2.1`.

### Steg 5️⃣ – Spara namn och adresser i en fil

Kommandot skapar filen `fwlab-vars.sh` med alla namn och IP-adresser som labben använder. Filen innehåller också funktionen `fwtest`, som du senare använder för att testa trafiken från den virtuella maskinen.

```bash
cat > ~/fwlab-vars.sh <<'EOF'
RG=rg-fwlab
LOC=swedencentral
VNET=vnet-fwlab
VNET_PREFIX=10.0.0.0/16
SUBNET_FW=AzureFirewallSubnet
SUBNET_FW_PREFIX=10.0.1.0/26
SUBNET_APP=snet-app
SUBNET_APP_PREFIX=10.0.2.0/24
VM=vm-app01
VM_IP=10.0.2.4
VM_SIZE=Standard_B2s
VM_IMAGE=Canonical:ubuntu-24_04-lts:server:latest
AFW=fw-fwlab
AFW_PIP=pip-fw-fwlab
AFW_IPCONF=fw-ipconfig
AFW_PRIVATE_IP=10.0.1.4
RT=rt-app
fwtest() {
  az vm run-command invoke -g "$RG" -n "$VM" --command-id RunShellScript \
    --scripts @"$HOME/fwlab-test.sh" --query "value[0].message" -o tsv
}
EOF
```

💡 **Förväntat resultat:** Inget skrivs ut. Prompten kommer tillbaka.

### Steg 6️⃣ – Skapa testskriptet

Skriptet körs senare inne på den virtuella maskinen. Det kontrollerar DNS, försöker nå fem webbplatser över HTTPS och hämtar till sist `http://www.facebook.com` över okrypterad HTTP.

```bash
cat > ~/fwlab-test.sh <<'EOF'
#!/bin/bash
echo "DNS: $(getent hosts www.wikipedia.org > /dev/null && echo OK || echo FEL)"
for url in https://www.wikipedia.org https://www.microsoft.com https://www.facebook.com https://www.instagram.com https://www.google.com; do
  code=$(curl -s -o /dev/null -w "%{http_code}" --max-time 15 "$url")
  rc=$?
  if [ "$rc" -eq 0 ]; then
     echo "TILLATEN $url (HTTP $code)"
  else
     echo "BLOCKERAD $url (curl-fel $rc)"
  fi
done
echo "Svar pa http://www.facebook.com:"
curl -s --max-time 15 http://www.facebook.com | head -c 300
echo
EOF
```

💡 **Förväntat resultat:** Inget skrivs ut. Prompten kommer tillbaka.

### Steg 7️⃣ – Läs in variablerna

```bash
source ~/fwlab-vars.sh
```

💡 **Förväntat resultat:** Inget skrivs ut.

### Steg 8️⃣ – Kontrollera variablerna

```bash
echo "$RG $LOC $SUBNET_APP_PREFIX $VM_IP $AFW_PRIVATE_IP"
```

💡 **Förväntat resultat:**

```text
rg-fwlab swedencentral 10.0.2.0/24 10.0.2.4 10.0.1.4
```

> 💡 **Tips:** Cloud Shell stängs efter en stunds inaktivitet och då försvinner variablerna. Om ett kommando plötsligt klagar på att ett namn saknas, kör stegen **Spara namn och adresser i en fil**, **Skapa testskriptet** och **Läs in variablerna** igen.

### Steg 9️⃣ – Kontrollera att VM-storleken finns i regionen

```bash
az vm list-skus -l $LOC --size $VM_SIZE --resource-type virtualMachines --query "[?name=='$VM_SIZE'] | [0].restrictions" -o json
```

💡 **Förväntat resultat:** `[]` – storleken är tillgänglig.

Visas en lista med begränsningar, eller `null`, ändrar du raden `VM_SIZE=Standard_B2s` i filen `fwlab-vars.sh` till:

```bash
VM_SIZE=Standard_B2ats_v2
```

Läs sedan in filen igen och kör kontrollen en gång till.

---

## 🌐 Del 1 – Skapa resursgruppen

Alla resurser i labben hamnar i resursgruppen `rg-fwlab` i regionen Sweden Central.

### Steg 1️⃣ – Skapa resursgruppen

```bash
az group create -n $RG -l $LOC -o table
```

💡 **Förväntat resultat:** En rad med `rg-fwlab`, `swedencentral` och `Succeeded`.

---

## 🔗 Del 2 – Skapa virtuellt nätverk och subnät

Nätverket `vnet-fwlab` (`10.0.0.0/16`) får två subnät. Brandväggens subnät måste heta exakt `AzureFirewallSubnet` och vara minst `/26`. Den virtuella maskinen hamnar i `snet-app` (`10.0.2.0/24`).

`snet-app` skapas som ett privat subnät med `--default-outbound-access false`. Det betyder att maskinerna där inte får någon automatisk väg ut till internet.

### Steg 1️⃣ – Skapa nätverket och brandväggens subnät

```bash
az network vnet create -g $RG -n $VNET -l $LOC \
  --address-prefixes $VNET_PREFIX \
  --subnet-name $SUBNET_FW --subnet-prefixes $SUBNET_FW_PREFIX \
  -o none
```

💡 **Förväntat resultat:** Inget skrivs ut när kommandot lyckas.

### Steg 2️⃣ – Skapa subnätet för den virtuella maskinen

```bash
az network vnet subnet create -g $RG --vnet-name $VNET -n $SUBNET_APP \
  --address-prefixes $SUBNET_APP_PREFIX \
  --default-outbound-access false \
  -o none
```

💡 **Förväntat resultat:** Inget skrivs ut när kommandot lyckas.

### Steg 3️⃣ – Kontrollera subnäten

```bash
az network vnet subnet list -g $RG --vnet-name $VNET --query "[].{namn:name, prefix:addressPrefix || addressPrefixes[0], standardUtgaende:defaultOutboundAccess}" -o table
```

💡 **Förväntat resultat:**

- `AzureFirewallSubnet` med prefixet `10.0.1.0/26`.
- `snet-app` med prefixet `10.0.2.0/24` och `False` i kolumnen `StandardUtgaende`.

---

## 🖥️ Del 3 – Skapa den virtuella maskinen

Maskinen `vm-app01` får den fasta adressen `10.0.2.4`. Den får ingen publik IP-adress och ingen NSG. Du styr den helt via kommandot `az vm run-command`, så du behöver varken SSH eller Bastion.

### Steg 1️⃣ – Skapa den virtuella maskinen

```bash
az vm create -g $RG -n $VM -l $LOC \
  --image $VM_IMAGE \
  --size $VM_SIZE \
  --vnet-name $VNET --subnet $SUBNET_APP \
  --private-ip-address $VM_IP \
  --public-ip-address "" \
  --nsg "" \
  --admin-username azureuser \
  --generate-ssh-keys \
  -o table
```

💡 **Förväntat resultat:** Efter några minuter visas en rad med `VM running`, private IP `10.0.2.4` och en tom kolumn för public IP.

### Steg 2️⃣ – Kontrollera maskinen

```bash
az vm show -g $RG -n $VM -d --query "{status:powerState, privatIp:privateIps, publiktIp:publicIps}" -o table
```

💡 **Förväntat resultat:** `VM running`, `10.0.2.4` och ingen publik IP.

---

## 🧪 Del 4 – Testa före brandväggen

Nu kör du testskriptet på den virtuella maskinen. Varje körning tar ungefär en till två minuter.

### Steg 1️⃣ – Kör testet

```bash
fwtest
```

💡 **Förväntat resultat:**

- Utskriften börjar med `Enable succeeded:` och `[stdout]`.
- Därefter visas `DNS: OK`.
- Alla fem adresser visar `BLOCKERAD` med curl-fel `28`.
- Efter raden `Svar pa http://www.facebook.com:` kommer en tom rad.

**Varför?** Subnätet är privat och det finns ännu ingen väg ut. Paketen släpps utan svar och curl ger upp efter 15 sekunder. Felkod `28` betyder timeout.

DNS fungerar ändå eftersom maskinen frågar Azures egen DNS på `168.63.129.16`, som nås inne i plattformen utan internet. Att namnuppslagning fungerar betyder alltså inte att maskinen når internet.

---

## 🛡️ Del 5 – Skapa Azure Firewall

Brandväggen skapas i tre steg: själva brandväggen, en publik IP och en IP-konfiguration som kopplar ihop brandväggen med `AzureFirewallSubnet`. Reglerna skrivs direkt på brandväggen.

### Steg 1️⃣ – Skapa brandväggens publika IP

```bash
az network public-ip create -g $RG -n $AFW_PIP -l $LOC \
  --sku Standard --allocation-method Static -o none
```

💡 **Förväntat resultat:** Inget skrivs ut när kommandot lyckas.

### Steg 2️⃣ – Skapa brandväggen

```bash
az network firewall create -g $RG -n $AFW -l $LOC \
  --sku AZFW_VNet --tier Standard -o none
```

💡 **Förväntat resultat:** Inget skrivs ut när kommandot lyckas.

### Steg 3️⃣ – Koppla brandväggen till nätverket

Det här steget tar lång tid, ofta tio minuter eller mer. Låt Cloud Shell-fliken vara öppen.

```bash
az network firewall ip-config create -g $RG -f $AFW -n $AFW_IPCONF \
  --public-ip-address $AFW_PIP --vnet-name $VNET -o none
```

💡 **Förväntat resultat:** Inget skrivs ut när kommandot lyckas.

> 💡 **Bra att veta:** Om Cloud Shell kopplas ner under väntan, öppna Cloud Shell igen, läs in variablerna enligt Del 0 och gå vidare till nästa steg. Driftsättningen fortsätter i Azure även om Cloud Shell stängs.

### Steg 4️⃣ – Uppdatera brandväggen

```bash
az network firewall update -g $RG -n $AFW -o none
```

💡 **Förväntat resultat:** Inget skrivs ut när kommandot lyckas.

### Steg 5️⃣ – Kontrollera brandväggen

```bash
az network firewall show -g $RG -n $AFW --query "{namn:name, status:provisioningState, niva:sku.tier, privatIp:ipConfigurations[0].privateIPAddress}" -o table
```

💡 **Förväntat resultat:** `fw-fwlab`, `Succeeded`, `Standard` och `10.0.1.4`.

> ⚠️ **Viktigt:** Står det något annat än `10.0.1.4` under `PrivatIp`, ändrar du raden `AFW_PRIVATE_IP=10.0.1.4` i filen till adressen som visas och läser in filen igen. Står det `Updating`, väntar du en minut och kör kommandot igen.

---

## 🛣️ Del 6 – Skicka trafiken via brandväggen

En route-tabell kopplad till `snet-app` skickar all trafik till `0.0.0.0/0` till brandväggens privata IP `10.0.1.4`. Det är brandväggen som nu blir maskinens väg ut.

### Steg 1️⃣ – Skapa route-tabellen

```bash
az network route-table create -g $RG -n $RT -l $LOC \
  --disable-bgp-route-propagation true -o none
```

💡 **Förväntat resultat:** Inget skrivs ut när kommandot lyckas.

### Steg 2️⃣ – Skapa standardrouten mot brandväggen

```bash
az network route-table route create -g $RG --route-table-name $RT -n udr-default-to-fw \
  --address-prefix 0.0.0.0/0 \
  --next-hop-type VirtualAppliance \
  --next-hop-ip-address $AFW_PRIVATE_IP \
  -o none
```

💡 **Förväntat resultat:** Inget skrivs ut när kommandot lyckas.

### Steg 3️⃣ – Koppla route-tabellen till `snet-app`

```bash
az network vnet subnet update -g $RG --vnet-name $VNET -n $SUBNET_APP \
  --route-table $RT -o none
```

💡 **Förväntat resultat:** Inget skrivs ut när kommandot lyckas.

> ⚠️ **Viktigt:** Koppla aldrig route-tabellen till `AzureFirewallSubnet`.

### Steg 4️⃣ – Hämta nätverkskortets ID

```bash
NIC_ID=$(az vm show -g $RG -n $VM --query "networkProfile.networkInterfaces[0].id" -o tsv)
```

💡 **Förväntat resultat:** Inget skrivs ut.

### Steg 5️⃣ – Kontrollera routes för maskinen

```bash
az network nic show-effective-route-table --ids $NIC_ID --query "value[?addressPrefix[0]=='0.0.0.0/0'].{kalla:source, status:state, nastaHopp:nextHopType, ip:nextHopIpAddress[0]}" -o table
```

💡 **Förväntat resultat:** En rad med `User`, `Active`, `VirtualAppliance` och `10.0.1.4`.

Finns det också en rad med `Default` och `Internet` ska den ha statusen `Invalid`.

### Steg 6️⃣ – Kör testet igen

```bash
fwtest
```

💡 **Förväntat resultat:**

- `DNS: OK`.
- Alla fem adresser visar fortfarande `BLOCKERAD`.
- Efter raden `Svar pa http://www.facebook.com:` kommer normalt ett svar från brandväggen som innehåller `Action: Deny`.

**Varför?** Trafiken når nu brandväggen, men den har inga regler. Azure Firewall nekar allt som inte uttryckligen är tillåtet. Svaret på HTTP-förfrågan kommer från brandväggen själv och bevisar att routen fungerar.

---

## 🚧 Del 7 – Skapa applikationsregler

Du skapar två regelsamlingar:

- `app-deny` har prioritet `100` och blockerar Facebook och Instagram.
- `app-allow` har prioritet `200` och tillåter ett fåtal utvalda webbplatser.

Lägst prioritetsnummer behandlas först, och den första regeln som matchar avgör.

Brandväggen läser domännamnet ur HTTP-huvudet `Host` för okrypterad trafik och ur SNI i TLS-handskakningen för HTTPS.

### Steg 1️⃣ – DENY, prioritet 100

```bash
az network firewall application-rule create -g $RG -f $AFW \
  --collection-name app-deny --name deny-meta \
  --action Deny --priority 100 \
  --protocols Http=80 Https=443 \
  --source-addresses $SUBNET_APP_PREFIX \
  --target-fqdns 'facebook.com' '*.facebook.com' 'instagram.com' '*.instagram.com' '*.fbcdn.net' \
  -o none
```

💡 **Förväntat resultat:** Raden `WARNING: Creating rule collection 'app-deny'`. Efter några minuter kommer prompten tillbaka.

### Steg 2️⃣ – ALLOW, prioritet 200

```bash
az network firewall application-rule create -g $RG -f $AFW \
  --collection-name app-allow --name allow-web-some \
  --action Allow --priority 200 \
  --protocols Http=80 Https=443 \
  --source-addresses $SUBNET_APP_PREFIX \
  --target-fqdns 'wikipedia.org' '*.wikipedia.org' '*.microsoft.com' 'aka.ms' '*.ubuntu.com' \
  -o none
```

💡 **Förväntat resultat:** Raden `WARNING: Creating rule collection 'app-allow'`. Efter några minuter kommer prompten tillbaka.

### Steg 3️⃣ – Kontrollera regelsamlingarna

```bash
az network firewall application-rule collection list -g $RG -f $AFW --query "[].{samling:name, prioritet:priority, atgard:action.type, antalRegler:length(rules)}" -o table
```

💡 **Förväntat resultat:**

- `app-deny` med `100`, `Deny` och `1` regel.
- `app-allow` med `200`, `Allow` och `1` regel.

### Steg 4️⃣ – Kör testet

```bash
fwtest
```

💡 **Förväntat resultat:**

- `TILLATEN https://www.wikipedia.org` med en HTTP-kod i 200- eller 300-serien.
- `TILLATEN https://www.microsoft.com` med en HTTP-kod i 200- eller 300-serien.
- `BLOCKERAD` för Facebook, Instagram och Google.
- Svaret på `http://www.facebook.com` innehåller normalt `Action: Deny`.

**Varför?** Facebook och Instagram stoppas av `app-deny`. Google finns inte i någon regel och stoppas därför av brandväggens standardregel som nekar allt annat. Blockerad HTTPS syns som ett curl-fel eftersom brandväggen inte kan skicka en läsbar felsida inuti en krypterad anslutning.

---

## ➕ Del 8 – Lägg till regler i en befintlig samling

När en samling redan finns lägger du till fler regler i den. Då ska du inte ange `--action` eller `--priority` – de hör till samlingen, inte till regeln.

### Steg 1️⃣ – Tillåt Google

```bash
az network firewall application-rule create -g $RG -f $AFW \
  --collection-name app-allow --name allow-google \
  --protocols Http=80 Https=443 \
  --source-addresses $SUBNET_APP_PREFIX \
  --target-fqdns 'www.google.com' \
  -o none
```

💡 **Förväntat resultat:** Inget skrivs ut när kommandot lyckas.

### Steg 2️⃣ – Försök tillåta Facebook i allow-samlingen

```bash
az network firewall application-rule create -g $RG -f $AFW \
  --collection-name app-allow --name allow-facebook \
  --protocols Http=80 Https=443 \
  --source-addresses $SUBNET_APP_PREFIX \
  --target-fqdns 'www.facebook.com' \
  -o none
```

💡 **Förväntat resultat:** Inget skrivs ut när kommandot lyckas.

### Steg 3️⃣ – Kontrollera samlingarna

```bash
az network firewall application-rule collection list -g $RG -f $AFW --query "[].{samling:name, prioritet:priority, atgard:action.type, antalRegler:length(rules)}" -o table
```

💡 **Förväntat resultat:** `app-allow` har nu `3` regler.

### Steg 4️⃣ – Kör testet

```bash
fwtest
```

💡 **Förväntat resultat:**

- `TILLATEN https://www.google.com`.
- `BLOCKERAD https://www.facebook.com` – fortfarande.

**Varför?** Regeln `allow-facebook` finns, men `app-deny` har prioritet `100` och behandlas före `app-allow` med prioritet `200`. Facebook matchar deny-regeln först och då letar brandväggen inte vidare.

Eftersom `100` är den högsta prioritet som finns kan ett undantag för Facebook bara göras genom att flytta deny-samlingen till ett högre nummer och lägga undantaget före den.

---

## 🧹 Del 9 – Städa upp

Azure Firewall kostar pengar för varje timme den finns. Ta bort resursgruppen direkt när du är klar.

### Steg 1️⃣ – Ta bort resursgruppen

```bash
az group delete -n $RG --yes --no-wait
```

💡 **Förväntat resultat:** Inget skrivs ut. Borttagningen fortsätter i bakgrunden.

### Steg 2️⃣ – Kontrollera borttagningen

Vänta tio minuter och kör sedan kommandot:

```bash
az group exists -n $RG
```

💡 **Förväntat resultat:** `false`.

Visas `true` pågår borttagningen fortfarande. Vänta några minuter och kör kommandot igen.

### Steg 3️⃣ – Ta bort labbfilerna i Cloud Shell

```bash
rm -f ~/fwlab-vars.sh ~/fwlab-test.sh
```

💡 **Förväntat resultat:** Inget skrivs ut.

---

## 🧠 Reflektionsfrågor

1. I Del 4 fungerade DNS men inte curl. Förklara varför och vad det säger om att använda `nslookup` som bevis för att en maskin når internet.
2. Varför var alla adresser fortfarande blockerade i Del 6, trots att routen mot brandväggen fungerade?
3. Regeln `allow-facebook` i Del 8 hade ingen effekt. Beskriv hur du skulle ändra prioriteterna för att göra ett undantag för `www.facebook.com`.
4. Google stoppades i Del 7 trots att ingen regel nämnde Google. Vilken princip ligger bakom, och varför är den bra ur säkerhetssynpunkt?
5. Blockerad HTTPS gav ett curl-fel, medan blockerad HTTP gav ett textsvar från brandväggen. Varför skiljer det sig?
6. Microsoft rekommenderar Firewall Policy i stället för regler direkt på brandväggen i produktion. Ta reda på vilka fördelar det ger.

---

## 🛠️ Felsökning

### `ERROR: Please run 'az login' to setup account.`

Cloud Shell har tappat inloggningen. Kör `az login`, följ instruktionen och läs sedan in variablerna igen:

```bash
source ~/fwlab-vars.sh
```

### Ett kommando klagar på att ett namn saknas eller är tomt

Variablerna har försvunnit, till exempel för att Cloud Shell startades om. Kör stegen **Spara namn och adresser i en fil**, **Skapa testskriptet** och **Läs in variablerna** i Del 0 igen.

### Kommandot `az network firewall` känns inte igen

Tillägget `azure-firewall` saknas. Kör steg 3 i Del 0 igen:

```bash
az extension add --name azure-firewall --upgrade -y
```

### `usage error: --collection-name EXISTING_NAME | --collection-name NEW_NAME --priority INT --action ACTION`

Du har angett `--priority` och `--action` för en samling som redan finns, eller glömt dem för en ny samling. Ta bort eller lägg till de två parametrarna enligt Del 8.

### `RequestDisallowedByPolicy` eller `SkuNotAvailable`

Prenumerationen tillåter inte regionen eller VM-storleken.

- Byt VM-storlek enligt det sista steget i Del 0.
- Gäller felet regionen: ta bort resursgruppen, ändra `LOC=swedencentral` i filen till en region som prenumerationen tillåter, till exempel `LOC=northeurope`, och börja om från Del 1.

### `AnotherOperationInProgress` eller status `Updating`

Brandväggen håller på med en tidigare ändring. Vänta två minuter och kör samma kommando igen.

### `fwtest` visar `BLOCKERAD` för Wikipedia efter Del 7

- Kör kontrollen av regelsamlingarna i Del 7. Båda samlingarna ska finnas.
- Kör kontrollen av routes i Del 6. Routen till `10.0.1.4` ska vara `Active`.
- Kontrollera att brandväggen har den privata adressen `10.0.1.4` enligt Del 5.

### `fwtest` svarar med `Conflict` eller att en körning redan pågår

Den förra testkörningen är inte klar. Vänta en minut och kör `fwtest` igen.

---

## 🔗 Källor

- [Deploy and configure Azure Firewall using Azure CLI](https://learn.microsoft.com/azure/firewall/deploy-cli)
- [Configure Azure Firewall rules – rule processing logic](https://learn.microsoft.com/azure/firewall/rule-processing)
- [Default outbound access in Azure](https://learn.microsoft.com/azure/virtual-network/ip-services/default-outbound-access)
- [az network firewall – tillägget azure-firewall](https://learn.microsoft.com/cli/azure/network/firewall)

---

**🎓 Klart!**

Du har nu byggt ett Azure-nätverk med en privat Linuxmaskin, skickat trafiken via Azure Firewall och testat hur deny-, allow- och standardregler påverkar åtkomsten till internet.
