# Plano: TV Samsung QN90C e box MEO (Sala), v2

14-10-2026. Substitui a v1, que foi escrita antes de ler o vault.

O que mudou em relação à v1:

- Respeita a regra **interface antes de YAML**: o Universal Media Player saiu, porque obrigava a editar o `configuration.yaml`.
- Respeita a regra **perguntar antes de remover**: nada é apagado sem verificar primeiro o que depende da entidade.
- Usa o mapeamento de entidades que está no vault.
- Parte do sintoma real que descreveste: o telemóvel funciona sempre; o HA liga a TV, mas a box só arranca às vezes, e falha quando esteve muito tempo desligada.

Legenda: **[V]** verificado (vault, leitura do HA ou fonte citada), **[I]** inferido.

---

## 1. Diagnóstico

### 1.1 Que integração está por trás de cada entidade

Feito a partir de `Nome_ID_Entidades_HA` (16-07-2026) e da leitura do HA de hoje.

| Entidade | `entity_id` | Integração | Exposta | Notas |
|---|---|---|---|---|
| Samsung HUB | `media_player.samsung_hub` + `remote.samsung_hub` | Samsung Smart TV, local **[I, forte]** (cria sempre um par media_player + remote) | Sim | Local e com push. É a candidata a entidade principal |
| Samsung HUB | `media_player.samsung_hub_2` + `remote.samsung_hub_2` | Samsung Smart TV, segunda entrada **[I]** | Sim | `unavailable` nas duas leituras **[V]**. Parece órfã |
| TV Samsung QN90CA | `media_player.tv_samsung_qn90ca_55` | SmartThings **[I, forte]** (tem sensores de energia, potência, canal e poupança de energia) | Sim | Volume diferente do da local (0,17 contra 0,12) **[V]** |
| Samsung QN90CA 55 | por confirmar | DLNA DMR **[I]** | Sim | Não existia a 16-07. `unavailable` com a TV desligada, `idle` com ela ligada **[V]** |
| Samsung QN90CA 55 Ligar | `scene.samsung_qn90ca_55_ligar` | Cena da SmartThings **[I]** | Sim | |
| Box MEO | `media_player.meobox_4k_diw377` + `remote.meobox_4k_diw377` | Android TV Remote **[I, forte]** | Sim | O modelo é a **MEO Box 4K DIW377** (Sagemcom, Android TV) **[V]** |
| Media MEO BOX | `media_player.media_meo_box` | Google Cast **[I, forte]** | Sim | |
| Ping TV Samsung / Ping Box MEO | `binary_sensor.tv_samsung` / `binary_sensor.ping_box_meo` | Ping | Não | **São sentinelas da deteção de falha de energia. Não se tocam** **[V]** |
| liga a tv / desliga a tv | `button.pedro_nogueira_design_gmail_com_liga_a_tv` / `..._desliga_a_tv` | Rotinas da Alexa (`alexa_devices`) **[I]** | Não | Há um caminho para a TV via Alexa que não está documentado |
| MEO Power | `script.meo_power` | Script | Não | Não consigo ler o conteúdo pelo MCP |
| Comandos Infravermelhos PIR | `switch.comando_infravermelhos` | ZHA, emissor IR Tuya ZS06 **[I]** | Retirado a 13-10 | O inventário diz que foi comprado precisamente para controlar a TV Samsung e a box MEO **[V]** |

**Para a mesma TV há cinco entidades expostas à Assist e ao MCP.** É por isso que um pedido de voz ou um comando ao MCP pode atuar sobre a entidade errada.

### 1.2 Porque é que a box falha

- **O que observei hoje:** às 09:17, com tudo desligado, as três entidades da box estavam `unavailable` **[V]**. A TV, pelo contrário, estava `off` e continuava a responder.
- **O que a MEO documenta para a box Android TV mais recente (HP4C02):** tem um standby de baixo consumo, "passive standby < 0,5 W". Configura-se em *Definições > Preferências do dispositivo > Alimentação e energia > Gestão energética* **[V, fórum MEO]**.
- **A minha leitura:** a DIW377 deve ter um modo equivalente **[I]**. Ao fim de algum tempo em standby passa para o standby passivo e desliga a rede. A partir daí o HA deixa de a ver e não a consegue ligar. Explica o "se esteve muito tempo desligada, já não liga".
- **O caminho atual é frágil por natureza:** hoje o HA liga a TV e espera que a TV acorde a box por HDMI-CEC. No sentido TV → box, cada fabricante implementa o CEC à sua maneira. No sentido inverso (a box liga e acorda a TV), o comportamento é o normal do CEC (*One Touch Play*) e é fiável.

### 1.3 Outros achados do vault com impacto aqui

1. **A TV (.190) e a box (.189) têm IP fixo configurado no próprio aparelho, mas esses endereços estão dentro da gama DHCP do router (.100 a .200)** **[V]**. A entrada de `decisions` de 30-07-2026 diz que estão fora da gama, e não estão. O router pode entregar o mesmo IP a outro aparelho, e se isso acontecer a TV ou a box ficam inacessíveis de forma intermitente. O aspirador S20 (.187) está na mesma situação.
2. **O `wake_on_lan:` já existe no `configuration.yaml`** e já há uma automação a usá-lo ("Wake-On-LAN - TV Suite") **[V]**. A automação de WOL para a TV da Sala faz-se toda pela interface, sem tocar no YAML.
3. **A TV e a box estão ligadas por Wi-Fi** **[V, auditoria]**. Isto torna o WOL da TV menos garantido do que por cabo. Ainda assim, ligar a TV já funciona hoje.
4. **Mudança de router (decisão de 13-10):** a ligação local da Samsung não passa entre sub-redes **[V, docs HA]**, o pacote de WOL é broadcast e também não passa, e a descoberta do Cast e do Android TV usa mDNS. A TV tem de ficar na mesma rede que o HA. Se a box ficar no router MEO (decisão de 30-07), o Cast e o Android TV têm de ser configurados por IP.

---

## 2. Proposta

A ideia é inverter a cadeia: **o HA liga a box, e a box acorda a TV por CEC.** Liga-se também a TV diretamente, como segurança. Para isso a box tem de continuar visível na rede quando está em standby.

### Passo 1: medir antes de mexer (5 min, só leitura)

No HA, em *Histórico*, ver os últimos 7 dias de `media_player.meobox_4k_diw377` e de `binary_sensor.ping_box_meo`. Medir quanto tempo passa entre a box ficar `off` e ficar `unavailable`. Esse intervalo é o temporizador do standby passivo.

Na mesma vista, ver se o `ping_box_meo` cai todas as noites. Se cair, a box pode também estar a contaminar as sentinelas "Infra" da deteção de falha de energia **[I]**.

(O Claude Code no PC de casa consegue tirar isto pela REST, com o `HA_TOKEN`.)

### Passo 2: a box deixa de desaparecer da rede

Na box: *Definições > Preferências do dispositivo > Alimentação e energia > Gestão energética*, escolher **Normal** ou a opção que não seja de baixo consumo. Este é o caminho da HP4C02; na DIW377 deve ser igual ou parecido **[I]**.

- **Custo:** mais consumo em standby. Não encontrei o valor para a DIW377.
- **Teste:** desligar a box e confirmar no HA que fica `off`, e não `unavailable`, ao fim de 10 min, 1 h e uma noite.
- **Plano B, se a opção não existir:** usar o emissor IR ZS06 que já está em casa. O IR acorda a box em qualquer modo de standby, mas obriga a aprender os códigos do comando no ZHA, o que dá mais trabalho. Antes disso é preciso confirmar se o comando da DIW377 é IR ou Bluetooth **[I]**.

### Passo 3: ligar a TV de forma explícita (interface)

Desde o HA 2026.8, o WOL implícito da integração Samsung está obsoleto **[V]**.

1. Copiar a automação "Wake-On-LAN - TV Suite" e trocar o MAC e a entidade. O trigger é **Device is requested to turn on** no dispositivo Samsung HUB (`samsung_hub`); a ação é **Wake on LAN: send magic packet** com o MAC Wi-Fi da TV.
2. O MAC da TV está na página do dispositivo no HA ou na lista de clientes do router.
3. Na TV, *Power On with Mobile* tem de estar ativo (*Geral > Rede > Definições avançadas*; os nomes dos menus em PT podem variar **[I]**).
4. Também na TV: Anynet+ (HDMI-CEC) ligado. Na box: CEC ligado.

### Passo 4: dois scripts expostos (editor visual)

| Script | Sequência |
|---|---|
| **Ver TV** | 1. `media_player.turn_on` em `meobox_4k_diw377`: a box arranca e acorda a TV por CEC. 2. `media_player.turn_on` em `samsung_hub`: segurança, por WOL. 3. Esperar até `samsung_hub` ficar `on` (máx. 30 s). 4. `media_player.select_source` HDMI em `samsung_hub`. 5. Ler o estado de volta e notificar se a box não ficou `on` |
| **Desligar TV** | `media_player.turn_off` em `samsung_hub`. A box desliga-se por CEC; se não desligar, `media_player.turn_off` em `meobox_4k_diw377` |

Antes de criar, ver o que faz o `script.meo_power`. Se já fizer parte disto, aproveita-se.

### Passo 5: exposição à Assist e ao MCP (reversível, não apaga nada)

| Entidade | Ação |
|---|---|
| `media_player.samsung_hub` | Fica exposta. Renomear para **TV Sala**, com os aliases *TV* e *Televisão* |
| `media_player.meobox_4k_diw377` | Fica exposta, com o nome **Box MEO** |
| Scripts **Ver TV** e **Desligar TV** | Expor |
| `samsung_hub_2`, `tv_samsung_qn90ca_55`, DLNA, cena "Ligar", `media_meo_box`, `remote.meobox_4k_diw377` | **Retirar da exposição.** Continuam a funcionar no dashboard e nas automações |

O resultado são **4 entradas** expostas para a Sala, em vez de 8.

### Passo 6: limpeza, só depois de confirmares e verificar dependências

Para cada candidata, abrir *Definições > Entidades > (entidade) > Relacionados* e procurar o `entity_id` no YAML do dashboard antes de remover.

| Candidata | Proposta | Porque não remover já |
|---|---|---|
| `samsung_hub_2` (órfã) | Apagar a entrada da integração | Pode ser a entrada que está na automação de falha de energia ou no dashboard |
| DLNA DMR | Apagar e carregar em **Ignorar** na descoberta | Nenhuma razão conhecida para ficar |
| SmartThings na TV | **Manter** | Dá os sensores de energia e potência, e as rotinas da Alexa podem depender dela |
| Cena "Samsung QN90CA 55 Ligar" | Apagar depois de o passo 3 funcionar | |

### Passo 7: IP da TV e da box (separado deste plano, mas relacionado)

Encurtar a gama DHCP do router para .100 a .179, para tirar de lá os IPs fixos (.187, .189, .190).

**Antes de mudar, verificar** que nenhum aparelho com IP dado por DHCP entre .180 e .200 é usado pelo HA através do IP. Os pings das sentinelas do Candeeiro Entrada, do Candeeiro Sala e do repetidor Xiaomi são os primeiros a confirmar.

---

## 3. Ordem e testes

| # | Passo | Depende de | Teste de aceitação |
|---|---|---|---|
| 1 | Histórico da box e do ping | | Intervalo entre `off` e `unavailable` medido |
| 2 | Gestão energética da box | 1 | A box fica `off` (não `unavailable`) depois de uma noite |
| 3 | Automação WOL da TV | | 3 em 3 arranques a partir de standby longo |
| 4 | Scripts Ver TV e Desligar TV | 2, 3 | 5 em 5 arranques, incluindo de manhã depois de uma noite desligada |
| 5 | Exposição | 4 | No MCP, "liga a TV" atua sobre uma entidade só |
| 6 | Limpeza | Confirmação tua | `Relacionados` vazio antes de cada remoção |
| 7 | Gama DHCP | Verificação dos pings | Sentinelas sem alterações durante 24 h |

---

## 📌 Decisões a registar quando aprovares

- A cadeia de arranque da Sala passa a ser HA → box → CEC → TV, com WOL da TV como segurança.
- A gestão energética da box passa a Normal (ou plano B por IR).
- A entidade principal da TV da Sala é `media_player.samsung_hub` (local). A SmartThings fica só para os sensores.
- A entrada de `decisions` de 30-07-2026 tem um erro: os IPs .189 e .190 estão dentro da gama DHCP.

## Fontes

- Vault: CLAUDE, PROJECT, STATUS, decisions, progress, Auditoria de Segurança 13-10-2026, Nome_ID_Entidades_HA, Configuration YAML - Registo, Technical Handover, Inventário por Zonas, HA - Contexto e Análise Inicial
- Samsung Smart TV: https://www.home-assistant.io/integrations/samsungtv/
- Android TV Remote: https://www.home-assistant.io/integrations/androidtv_remote/
- WOL implícito obsoleto: https://github.com/home-assistant/core/issues/166226
- Gestão energética da MEOBox Android TV: https://forum.meo.pt/tv-e-pacotes-9/meobox-android-tv-4k-strong-hp4c02-165657
