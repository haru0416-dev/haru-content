---
title: "Codex が SKILL.md を 220 行で切る原因は、Codex 自身の prompt の 1 行だった"
emoji: "🔧"
type: "tech"
topics: ["codex", "claudecode", "ai", "agent", "llm"]
published: true
---

[前回の記事](https://zenn.dev/haru0416/articles/codex-skill-md-220-lines)で、Codex CLI が SKILL.md を 220 行で打ち切る現象を計測した。そのときの結論は「model が学習データから引いてきた『SKILL.md の見慣れた長さ ≈ 200 行』が cap として出ている」だった。

これは半分しか合っていなかった。本当の原因は、**Codex CLI 自身の prompt が「Read only enough to follow the workflow」と明示的に書いている**ことだった。openai/codex に既に Issue が立っていて、提案 patch まで作られている。今もそのままだ。前回書けなかった root cause と、それを裏付ける実測を、ここに置いておく。

## 犯人の 1 行

Codex CLI v0.128.0 (および 2026-05-28 時点の main HEAD `5314f550`) の `codex-rs/core-skills/src/render.rs` を読むと、skill が activate されたときに model に渡る prompt の中に次の行が入っている。

```rust
// codex-rs/core-skills/src/render.rs
pub const SKILLS_HOW_TO_USE_WITH_ABSOLUTE_PATHS: &str = r###"
- How to use a skill (progressive disclosure):
  1) After deciding to use a skill, open its `SKILL.md`.
     Read only enough to follow the workflow.
  ...
"###;
```

**「Read only enough to follow the workflow」** ── これだけ。skill 本文を「partial にしか読むな」と明示している。

これは仮説ではない。私の 2026-05-28 の Codex セッション (`rollout-2026-05-28T01-17-32-...jsonl`) を `grep` すると、この文字列が会話の中にそのまま入っていた。runtime で確認できる、文字通りの prompt 指示だ。

model はその「enough」をどう解釈するか? それぞれの世代で持っている prior で解決する:

- gpt-5.5 → だいたい **220 行**
- gpt-5.4 → だいたい **260 行**

前回の記事で観測したばらつきは、**同じ prompt 指示を、各 model が自分の prior で違う数字に解いた結果**だった。

## 最新版で直っているか

私が計測したのは v0.128.0 だけど、最新版を入れ直したら直るのでは、と気になる。確認した。

最新 release は **v0.134.0 (2026-05-26 公開)**。私の計測時点から 6 versions 進んでいる。`rust-v0.128.0` と `rust-v0.134.0` の render.rs を直接 diff すると、変更は `plugin_id: None,` の 1 行追加 (skill analytics 用のフィールド) だけ。**prompt 文字列は 1 文字も変わっていない**。

source level で確定するなら runtime も同じはずだが、実際に upgrade して回してみた。

- gpt-5.5: quaere-evidence と quaere-semantic は **220**、quaere-audit は **240** (旧と同じ揺れ方)
- gpt-5.4: quaere-audit は **260**、quaere-semantic も **260** (旧と同じ 260 軸)
- cap-test (prefix で「読め」と明示) は **二手目を打って marker 取得** (旧と同じ)
- chain (quaere-audit → external-grounding → quaere-execution) も発生 (旧と同じ)

挙動が変わった点はゼロ。Issue #16479 の wording が直されない限り、update では消えない。

## 既に Issue が立っていた

[openai/codex#16479](https://github.com/openai/codex/issues/16479) ── 2026-04-01 に hannesrudolph 氏が立てた issue。同じ問題を指摘していて、proposed wording まで書いてある:

> Current wording (step 1):
>
> > After deciding to use a skill, open its SKILL.md. Read only enough to follow the workflow.
>
> Suggested wording:
>
> > After deciding to use a skill, open and read its SKILL.md in full. This is the primary workflow definition. Then load additional resources (e.g. references/, scripts/, assets/) progressively as needed.

別の community 寄与者 cluedesc 氏が patch まで作っている ([`cluedesc:fix/skills-read-skill-md-first`](https://github.com/cluedesc/codex/tree/fix/skills-read-skill-md-first))。

ところが openai/codex は **invitation でしか PR を受け付けていない**。hannesrudolph 氏も cluedesc 氏もメンバーじゃないので、PR を出せない。Issue は OPEN のまま、4 コメントすべて外部の人 (`author_association: NONE`) で、OpenAI のチームからは何もコメントが出ていない。最終更新は 2026-04-06、**2 ヶ月近く動いていない**。

## 私の実測が Issue の主張を裏付けるか

#16479 は wording を変えるべきだと議論の側から指摘している。私の側からは、実際に runtime でそれが起きていることを計測で示せる。

数値はすべて手元で再現できる (gist にコマンドと生データを置いてある)。要点だけ:

### `~/.codex/sessions/` 全 209 セッションの cap 分布

71 セッションが SKILL.md を sed で読みにいき、計 143 個の read を出した。これを model と cap で並べると:

| model | cap=220 | cap=240 | cap=260 | その他 |
|---|---:|---:|---:|---:|
| gpt-5.5 | **80** (76%) | 10 | 11 | 4 (cap-test の段階読み) |
| gpt-5.4 | 9 | 2 | **27** (71%) | 0 |

**同じ prompt、同じ codex exec 経路、model だけ変えると分布が逆転する**。これは Issue が言う「the model has to guess what counts as 'enough'」を、そのまま実測で見せている。

### Harness を変えると cap が消える

通常 Claude Code は Claude に、Codex CLI は OpenAI のモデルに紐づいていて、組み合わせは固定されている。これを [claudex](https://github.com/EdamAme-x/claudex) でずらして、Claude Code の harness のまま gpt-5.5 を動かせる。同じ model を Codex CLI の prompt に晒さない経路で走らせる実験になる。

11 セッション回した。

- first move = `Skill(...)` tool 呼び出し: **11 / 11**
- `sed -n '1,Np' .../SKILL.md`: **0 / 11**

Codex CLI の prompt が無い経路では、同 model でも cap は **そもそも発生しない**。

### Skill 自身に「もっと読め」と書けば、model は読む

「cap-test」という skill を作って試した。200 行を超えた位置に Iron Law marker を置いてあって、**prefix の中で「`sed -n '221,$p'` で残りを読め」と明示**してある。

- gpt-5.5: **7 / 7** で marker 取得 (= 二手目を打って 221+ を読んだ)
- gpt-5.4: **4 / 4** で marker 取得

つまり model は本来 partial 読みに固執していない。Codex CLI の prompt が「partial で十分」と言っているから、そこで止まっているだけだ。

## 仕様の側はどう言っているか

[Agent Skills 仕様の Progressive disclosure](https://agentskills.io/specification#progressive-disclosure) は、こう書いている:

> "Note that the agent will load this entire file once it's decided to activate a skill."

> "Instructions (< 5000 tokens recommended): The full SKILL.md body is loaded when the skill is activated."

> "Keep your main SKILL.md under 500 lines."

つまり仕様が言っているのは「activation 時に本文を丸ごとロード」「上限 500 行 / 5000 tokens」。Codex CLI の prompt は「partial で十分」── これは正面から矛盾している。

前回の記事で書いた「だいたい 200 行に収まる」「だから 220 は仕様どおり」という説明は、仕様を読み間違えていた。500 行が許容ライン。220 はその半分以下で打ち切られている。

## 何が修正されればいいか

Issue #16479 の proposed wording で十分。1 行を:

```diff
- 1) After deciding to use a skill, open its `SKILL.md`. Read only enough to follow the workflow.
+ 1) After deciding to use a skill, open and read its `SKILL.md` in full. This is the primary workflow definition. Then load additional resources (e.g. `references/`, `scripts/`, `assets/`) progressively as needed.
```

ただし live main では同じ wording が 2 箇所 (`SKILLS_HOW_TO_USE_WITH_ABSOLUTE_PATHS` と `SKILLS_HOW_TO_USE_WITH_ALIASES`) にある。cluedesc patch (2026-04-02) はそのうち片方しか触っていない。2026-04-24 の `#19098 "Compress skill paths with root aliases"` で第 2 const が後から増えたせいだ。current main に rebase して 2 つとも直す手当が要る。

「Context hygiene」セクションの「Keep context small」「Avoid deep reference-chasing」も同じ方向の指示で、本気で直すなら一緒に見直す余地はある。ただし step 1 の修正で root cause の大半は塞がるはず。

## Quaere 側で言えること

前回の記事で「SKILL.md 本体は 200 行以内に収める」という助言を書いた。これは今でも実用上は使えるが、扱いは変わる:

- 仕様 (500 行) に合わせる助言ではなく、Codex CLI の prompt 由来の現場 cap に合わせる workaround。具体的な cap は gpt-5.5 で 220、gpt-5.4 で 260。
- 上流の wording が直れば、この助言は要らなくなる。
- Codex 以外の harness (Claude Code、OpenCode、claudex 経路) では最初から要らない。

Quaere の skill 群を 200 行以内に書き直す仕事は宿題として残っている。書き直してから再 eval すると、現在 README にある eval 数字 (+37.7pp / +8.7pp) が「Codex が後半を読まないまま走った結果」なのか「最初の 200 行で実質足りていた」のかが、ようやく分けて測れる。

## 計測のソース

raw data と再現手順は gist に置いてある。

- 計測ノート: https://gist.github.com/haru0416-dev/8c1b01098f46e29d244f2085e408c789
- English version: https://gist.github.com/haru0416-dev/4fd584e8616698cc1cc34c04b03f70ee

cap-test skill (実験用、prefix で「読め」と明示してある skill) と、Codex セッションログの直接 grep で確認できる。

## 前回からのつながり

[前回の記事](https://zenn.dev/haru0416/articles/codex-skill-md-220-lines)では、計測の生データだけを出した。「なぜ 220 なのか」の解釈は推測込みで、結果としていくつか間違いがあった。

今回見つかった root cause ── render.rs の 1 行 + 既存の Issue #16479 ── は、それをはっきり書き直してくれる。
