# Plano: TV Samsung QN90C e box MEO (Sala de Estar)

14-10-2026. O estado vem de três leituras ao HA via MCP: às 09:17 (TV e box desligadas) e às ~09:25 (TV e box ligadas, a dar TVI).

Legenda: **[V]** verificado (lido no HA ou na documentação oficial), **[I]** inferido.

---

## 1. Diagnóstico

### 1.1 O que o HA expõe hoje

Atenção: o MCP só mostra as entidades **expostas** ao Assist. Podem existir mais (por exemplo, a entidade `remote` da Samsung), mas essas não aparecem aqui.

| Entidade | Domínio | TV/box desligada | TV/box ligada | Leitura |
|---|---|---|---|---|
| TV Samsung QN90CA | media_player | off | on, volume 0,17 | Integração ativa. Volume diferente das outras duas **[V]** |
| Samsung HUB (1) | media_player | off | on, volume 0,12 | Integração ativa **[V]** |
| Samsung HUB (2) | media_player | unavailable | unavailable | Órfã: nunca responde **[V]**. Restos de uma integração antiga **[I]** |
| Samsung QN90CA 55 | media_player | unavailable | idle, volume 0,12, sem área nem device_class | Comportamento típico de DLNA Digital Media Renderer **[I]** |
| Samsung QN90CA 55 Ligar | scene | unknown | unknown | Cena da SmartThings ou cena manual **[I]** |
| Box MEO | media_player | unavailable | on, volume 0,44 | Android TV Remote (cria media_player + remote) **[I, forte]** |
| Comando Box | remote | unavailable | on | Android TV Remote **[I, forte]** |
| Media MEO BOX | media_player | unavailable | playing "Diário da Manhã / TVI", volume 0,0 | Google Cast **[I, forte]** |

### 1.2 Conclusões

1. **A TV aparece quatro vezes como media_player, mais uma cena.** O mais provável é estar a ser apanhada por três integrações e uma órfã: Samsung Smart TV (local), SmartThings, DLNA DMR e uma entrada antiga **[I]**. Não consigo ver pelo MCP qual entidade pertence a qual integração. Isso confirma-se no passo 0.
2. **Duas entidades "ativas" discordam no volume** (0,17 contra 0,12). Uma delas está desatualizada ou lê de outra fonte (cloud contra local) **[I]**. É precisamente o problema de ter várias integrações: o Assist ou o MCP pode agir sobre a errada.
3. **Às 09:17, com tudo desligado, as três entidades da box estavam `unavailable`, e não `off`** **[V]**. Isto quer dizer que a box sai da rede quando está em standby **[I]**. Daí a sensação de que "não funciona": o HA não consegue ligar um aparelho que deixou de ver.
4. **A box está bem montada em termos de integrações.** Android TV Remote e Google Cast complementam-se, e a própria documentação do HA recomenda usá-las juntas **[V]**. O que falta é juntá-las numa única entidade e resolver o standby.
5. **O Cast reporta volume 0,0** enquanto o Android TV Remote reporta 0,44 **[V]**. Não uses o Cast para o volume.
6. **Desde o HA 2026.8, a Samsung Smart TV marca como obsoleto o Wake-on-LAN implícito.** É preciso uma automação explícita para ligar a TV **[V]**. Se aparecer um aviso em *Settings > Repairs*, a causa é esta.

---

## 2. Arquitetura alvo

| Aparelho | Integração que fica | Para quê | O que sai |
|---|---|---|---|
| TV | **Samsung Smart TV (local)** | Estado imediato (push local), volume, fonte TV/HDMI, canais, teclas | DLNA DMR, órfã, cena "Ligar". SmartThings só fica se o WOL falhar (ver 4.3) |
| Box | **Android TV Remote + Google Cast**, juntas num **Universal Media Player** | ATV Remote: ligar, desligar, volume, teclas, canais. Cast: o que está a dar | Nada se remove: as duas entidades ficam, mas escondidas da exposição |

No fim ficam expostas **duas entidades** (TV Sala, Box MEO) e, se quiseres, dois scripts (Ver TV, Desligar TV). Hoje são oito.

---

## 3. Passo 0: inventário (5 min, só no HA)

1. *Settings > Devices & services > Entities*, pesquisar `samsung` e depois `meo`. Anotar a coluna **Integration** de cada entidade e o `entity_id`.
2. *Settings > Repairs*: ver se há aviso de "Implicit Wake-On-LAN is deprecated".
3. HACS: ver se o `samsungtv_smart` (ollo69) está instalado.
4. *Settings > Devices & services*: ver se a integração **SmartThings** é usada para outros aparelhos além da TV.

Manda-me esta lista e eu fecho o mapeamento exato.

---

## 4. TV Samsung QN90C

### 4.1 Rede e TV (pré-requisito)

- **IP fixo:** reserva DHCP no router para a TV e para a box.
- **A TV tem de estar na mesma sub-rede que o HA.** A Samsung não aceita WebSocket entre sub-redes ou VLANs **[V]**.
- Na TV, ativar **Ligar com dispositivo móvel** (*Power On with Mobile*). Fica em *Definições > Geral > Rede > Definições avançadas*; os nomes exatos dos menus em PT podem variar **[I]**.
- Na TV, em *Gestor de ligação de dispositivos > Notificação de acesso*: escolher **Apenas a primeira vez**. Na *Lista de dispositivos*, apagar as ligações antigas do HA **[V]**.
- Na TV, ativar o **Anynet+ (HDMI-CEC)**. É o que vai permitir que a TV acorde a box (secção 5.2).

### 4.2 Limpeza

1. Remover a entrada **DLNA Digital Media Renderer** da TV. Depois, no cartão "Discovered", carregar em **Ignore** para não voltar a aparecer.
2. Apagar a órfã **Samsung HUB (unavailable)**: abrir o dispositivo dela e apagá-lo.
3. Se o `samsungtv_smart` estiver no HACS, removê-lo. Não se deve misturar com a integração core.
4. Apagar a cena **Samsung QN90CA 55 Ligar** depois de a automação de 4.3 funcionar.
5. Renomear a entidade que fica (a da integração Samsung Smart TV) para **TV Sala**, com `entity_id` `media_player.tv_sala` e os aliases *Televisão* e *TV*.

### 4.3 Ligar a TV (substitui o WOL implícito)

Criar a partir da UI: abrir o dispositivo da TV, ir a *Automations > +*, escolher o trigger **Device is requested to turn on** e a ação **Wake on LAN: send magic packet**.

O equivalente em YAML, adaptado da documentação oficial:

```yaml
# configuration.yaml
wake_on_lan:

# automations.yaml
- alias: "TV Sala: turn on via WOL"
  triggers:
    - trigger: samsungtv.turn_on
      entity_id: media_player.tv_sala
  actions:
    - action: wake_on_lan.send_magic_packet
      data:
        mac: "AA:BB:CC:DD:EE:FF"   # MAC da TV (Ethernet ou Wi-Fi, conforme a ligação)
```

**Teste:** desligar a TV, esperar 10 min para ela entrar em standby profundo e ligar pelo HA. Repetir três vezes.

- **3 em 3 resultam:** remover a TV da SmartThings (ou a integração inteira, se não servir para mais nada).
- **Falha:** manter a SmartThings só como caminho para ligar. Trocar a ação da automação por `media_player.turn_on` na entidade da SmartThings e tirar essa entidade da exposição.

---

## 5. Box MEO (Android TV)

### 5.1 Uma entidade só (Universal Media Player)

Adaptado do exemplo oficial do Android TV Remote. Os `entity_id` são prováveis e têm de ser confirmados no passo 0.

```yaml
# configuration.yaml
media_player:
  - platform: universal
    name: Box MEO Sala
    unique_id: box_meo_sala
    device_class: tv
    children:
      - media_player.box_meo        # Android TV Remote
      - media_player.media_meo_box  # Google Cast
    browse_media_entity: media_player.media_meo_box
    commands:
      turn_on:
        action: media_player.turn_on
        data:
          entity_id: media_player.box_meo
      turn_off:
        action: media_player.turn_off
        data:
          entity_id: media_player.box_meo
      volume_up:
        action: media_player.volume_up
        data:
          entity_id: media_player.box_meo
      volume_down:
        action: media_player.volume_down
        data:
          entity_id: media_player.box_meo
```

Depois:

- Expor apenas **Box MEO Sala**.
- Tirar da exposição *Box MEO*, *Media MEO BOX* e *Comando Box*. O remote continua a funcionar em scripts.
- Para os canais, usar `media_player.play_media` com `media_content_type: channel` na entidade do ATV Remote **[V]**. Podes criar scripts de favoritos ("Pôr na RTP1"), por exemplo.

### 5.2 Standby (a causa provável do "não funciona")

**Teste:** desligar a box com o comando, esperar 10 min e ver no HA se *Box MEO* fica `off` ou `unavailable`.

- **Fica `off`:** a box mantém a rede ligada em standby e o HA consegue ligá-la. Fica resolvido.
- **Fica `unavailable`:** a box corta a rede em standby **[I]**. Há dois caminhos:
  1. **Nas definições de energia da box**, procurar um modo de poupança ou de standby de baixo consumo e desativá-lo. Não confirmei o nome do menu na box MEO.
  2. **Encadear pelo CEC:** o HA liga a TV (WOL), o Anynet+ acorda a box, e quando a TV desliga a box desliga também. Não depende da box estar visível na rede. É este que recomendo, porque funciona mesmo que o caminho 1 não exista.

Se o Android TV Remote pedir para emparelhar outra vez ou desligar-se a cada 15 s, a documentação oficial indica **[V]**:

- reiniciar a box depois do emparelhamento;
- se não chegar, ir a *Definições > Apps > Apps do sistema > Android TV Remote Service > Armazenamento > Limpar*, reiniciar o HA e emparelhar de novo.

### 5.3 Cast

- O Cast serve para os metadados (programa e canal) e para enviar conteúdo. Não serve para saber se a box está ligada **[V]** nem para o volume (reporta 0,0) **[V]**.
- Se o problema for fazer cast a partir do telemóvel, as causas habituais são o mDNS e o multicast no router, ou o telemóvel numa rede diferente da box (Wi-Fi de convidados, banda isolada) **[I]**. Preciso de saber o sintoma exato (ver perguntas).

---

## 6. Opcional: um comando para a sala toda

Dois scripts expostos ao Assist, à Alexa e ao MCP:

- **Ver TV:** `media_player.turn_on` em `tv_sala` (WOL), espera 10 s, e `media_player.select_source` HDMI (a entrada da box). A box acorda por CEC.
- **Desligar TV:** `media_player.turn_off` em `tv_sala`. A box desliga por CEC.

Assim "liga a TV" faz sempre o mesmo, venha o pedido de onde vier.

---

## 7. Ordem de execução

| # | Passo | Tempo | Depende de |
|---|---|---|---|
| 0 | Inventário | 5 min | |
| 1 | IP fixo, Power On with Mobile, Anynet+, notificação de acesso | 10 min | |
| 2 | Automação WOL e teste 3x | 15 min | 1 |
| 3 | Limpeza da TV (DLNA, órfã, HACS, cena, SmartThings conforme o teste) | 10 min | 2 |
| 4 | Teste de standby da box | 10 min | |
| 5 | Universal Media Player da box e exposição | 15 min | 0 |
| 6 | Scripts Ver TV / Desligar TV (opcional) | 10 min | 2, 4 |

## Fontes

- Samsung Smart TV: https://www.home-assistant.io/integrations/samsungtv/ (documentação da versão 2026.10.1)
- Android TV Remote: https://www.home-assistant.io/integrations/androidtv_remote/
- SmartThings, media player: https://www.home-assistant.io/integrations/smartthings/
- WOL implícito obsoleto (HA 2026.8.0): https://github.com/home-assistant/core/issues/166226
