---
name: coc-investigator
description: >-
  生成一名《克苏鲁的呼唤》第七版(CoC 7e)调查员(含背景故事)并归档到调查员咖啡馆
  (Investigator Café)。当用户想掷点 / 创建 CoC 调查员并保存到自己账号时使用。规则与
  字段规范从站点 API 实时读取,本 skill 不内置任何规则。
---

# CoC 7e 调查员建档 · Investigator Café

生成一名 CoC 7e 调查员并归档到 **investigator.cafe**。你(AI)全程通过站点 REST API
完成;规则(属性 / 技能目录)与**字段规范**(背景档案键、各类内容的归位端点)都从
`GET /rulesets/coc7` 实时获取,所以这里不保留任何规则表。

- **API 基址**:`https://investigator-archive-api.fly.dev/api/v1`
- **鉴权**:个人令牌放在环境变量 `INVESTIGATOR_CAFE_TOKEN`,每个写操作都带
  `Authorization: Bearer <token>`。

## 内容归位表(最重要:别把完整背景塞进 biography)

| 用户 / 你产出的内容 | 正确落位 |
|---|---|
| 一两段人物简介、外貌 | `character.biography`(**只放简短摘要**) |
| 结构化背景(思想信念、意义之地、特质、恐惧躁狂、创伤疤痕、典籍、第三类接触、消费 / 现金 / 资产) | `character.background`(对象,键取自 `background_fields`) |
| 第一人称经历、日记 | `POST /characters/{id}/journals` |
| 装备、线索、遗物、珍视之物 | `POST /characters/{id}/items` |
| 伙伴、亲属、恩人、敌人 | `POST /characters/{id}/relationships`(先建 NPC 再链接) |
| 跑团事件、结局 | `POST /characters/{id}/events`;结局用 `PATCH` 改 `status` |

> **硬规则**:`biography` 只能是**首页用的一两段简短简介**。**绝不**把思想信念、意义
> 之地、创伤、珍视之物等完整背景拼成长文塞进 `biography`;完整资料必须分别写入
> `background` / `items` / `relationships` / `journals`。

## 步骤

1. **拿令牌(若没有)。** 若未设置 `INVESTIGATOR_CAFE_TOKEN`,不要瞎猜——把下面这个
   链接给用户,请他打开、登录、生成令牌并粘回给你:

   > https://investigator.cafe/token

   之后所有请求都用这枚令牌作 Bearer。绝不臆造或打印令牌。令牌默认约 1 天过期——
   任何请求返回 **401** 即令牌失效,请用户到同一页面重取。

2. **读规则与字段规范** —— `GET /rulesets/coc7`。用它返回的:
   - `characteristics` —— 要写入的**全部数值槽**(STR…EDU 与派生 HP/MP/SAN/MOV/LUCK/DB/BUILD),
     各 `{key, display_name, roll}`;`roll` 是该项的**掷骰 / 推导公式**(如 STR `3D6×5`、
     SIZ `(2D6+6)×5`、HP `(CON+SIZ)/10`、SAN `=POW`、LUCK `3D6×5`);
   - `skills` —— 技能目录,各 `{skill_key, display_name, base, category, specialization}`;
   - `background_fields` —— **结构化背景对象的合法键**(如 `ideology / locations /
     traits / injuries / phobias / tomes / encounters / wealth / cash / assets`),各 `{key, display_name}`;
   - `record_types` —— 各类内容对应的端点与字段(background / journal / item /
     relationship / event)。
   一律照这些键来,别自己发明键名或规则。

3. **询问**(若未给)调查员的年龄(15–89)与一个概念 / 主题。

4. **掷出全部数值**(每一项都要写进档案),按 `characteristics` 里各项的 `roll` 公式:
   - **8 项属性**(掷骰):STR `3D6×5`、CON `3D6×5`、SIZ `(2D6+6)×5`、DEX `3D6×5`、
     APP `3D6×5`、INT `(2D6+6)×5`、POW `3D6×5`、EDU `(2D6+6)×5`。
   - **派生值**(推导):HP `(CON+SIZ)/10` 向下取整、MP `POW/5`、SAN 初始 `=POW`、
     幸运 LUCK `3D6×5`、MOV(由 STR/DEX/SIZ 与年龄定)、DB 与 BUILD(由 STR+SIZ 定)。
   套用年龄调整;以 ruleset 的 `roll` 为准,别自己改公式,算术要复核。

5. **分配技能**,在目录基础值之上:职业点池 = EDU×4(或该职业自己的公式),兴趣点池
   = INT×2。选一个贴合概念的职业;上限 99;克苏鲁神话建卡时不可加点。

6. **写原创背景**,与数值自洽。分别产出:一段**简短个人简介**(→ biography);结构化
   背景各项(→ background 各键);若剧情涉及,再有日记、珍视之物 / 装备、重要之人。全为原创文字。

7. **归档**(Bearer = 令牌),按内容归位表各就各位:
   - **先查重** —— `GET /me/characters` 返回 `{characters:[{id,name,…}]}`。若已有
     同名调查员,别闷头新建:问用户是要新建还是更新那一份。
   - `POST /characters` —— `{name, occupation, age, gender, birthplace, residence,
     era, biography, ruleset:"coc7", initial_san, initial_hp, initial_luck}`。
     **`biography` 只放简短简介。** 记下返回的 `id`。
   - `PUT /characters/{id}/attributes` —— `[{key, display_name, numeric_value,
     category:"characteristic", sort_order}]`,填 `characteristics` 列出的**每一个**属性槽。
   - `PUT /characters/{id}/skills` —— `[{skill_key, display_name, value}]`(用目录的
     键;只提交你分配了的技能)。
   - `PATCH /characters/{id}` —— `{ "background": { <background_fields.key>: "文本", … } }`。
     **结构化背景写这里,不是 biography。** 有哪项填哪项,其余省略。
   - **珍视之物 / 装备 / 遗物** → `POST /characters/{id}/items`
     `{name, item_type?, description?}`(每件一次)。
   - **重要之人** → `POST /characters/{id}/relationships`
     `{target_character_id, relationship_type?, description?}`。对象必须是站内角色:
     先 `POST /characters` 建那个 NPC 拿到 id 再链接;不想建 NPC 就并入 biography。
   - **日记 / 第一人称经历** → `POST /characters/{id}/journals`
     `{title?, body_markdown, written_at?}`。
   - **头像(可选)** —— `avatar` 是一张图片 URL,可在建档时随 `POST /characters` 带上,
     或事后 `PATCH /characters/{id}` `{avatar: "<图片URL>"}` 更新。
     - 用户给了图片 **URL** → 直接把它作为 `avatar`。
     - 用户要传**本地图片** → 先 `POST /media/upload`(multipart 表单,字段为图片文件,
       ≤4MB,png/jpg/webp/gif;可先 `GET /media/enabled` 确认已开启),它返回 `{url}`,
       再把这个 url 作为 `avatar` PATCH 上去。
   - **结局(可选)** → `PATCH /characters/{id}` `{status, death_cause, died_at}`,
     `status ∈ deceased/retired/missing/insane`;会自动生成时间线事件并归档。默认存活,别乱设。

8. **写后校验(必做)** —— `GET /characters/{id}`,确认:
   - `character.background` **不是空对象**(结构化背景确实写进去了);
   - `journals` / `items` / `relationships` 数量与你提交的一致;
   - `character.biography` 仍是**简短**摘要(没被塞成长文)。
   **不得仅凭 PATCH / POST 返回成功就宣称完成**——以 GET 结果为准。
   最后把档案链接给用户:`https://investigator.cafe/<id 前 8 位>`。

## 说明
- 规则 / 字段规范全来自 `GET /rulesets/coc7`(含 `background_fields` 与 `record_types`)——本 skill 不保留任何规则。
- 令牌是机密:从环境读取,绝不打印。
- 非官方同人工具,实现公开已知的 CoC 7e 建卡流程(仅机制,不含规则书原文)。
  《克苏鲁的呼唤》是 Chaosium Inc. 的商标;本 skill 与其无关、未获其背书。
- `EXAMPLE.md` 有一份可直接照搬的 API 调用范例。
