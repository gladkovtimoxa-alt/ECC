---
description: Сгенерировать локальный отчёт о расходах Claude Code из лога метрик ECC cost-tracker. Используйте для сводки в терминале или экспорта CSV отслеженных расходов; навык cost-tracking покрывает тот же лог метрик для вопросов о затратах/бюджете по запросу в разговоре.
argument-hint: "[csv]"
---

# Cost Report

Суммировать локальные расходы Claude Code по дням, моделям и сессиям из лога
метрик, который пишет хук ECC `stop:cost-tracker`.

## Где живут данные

Трекер добавляет один JSON-объект на каждую остановку сессии в
`~/.claude/metrics/costs.jsonl`. Каждая строка — это **кумулятивный снапшот для
этой сессии**, поэтому отчёт берёт **последнюю строку на `session_id`** и
суммирует по сессиям (суммирование каждой строки умножило бы счёт).

Схема строки:
`{ timestamp, session_id, transcript_path, model, input_tokens, output_tokens, cache_write_tokens, cache_read_tokens, estimated_cost_usd }`

## Что делает эта команда

1. Проверяет, что `~/.claude/metrics/costs.jsonl` существует. Если нет, говорит
   пользователю, что трекер ещё не настроен (он заполняется после завершения
   первой сессии с включённым хуком `stop:cost-tracker`).
2. Сводит строки к последнему снапшоту на сессию и агрегирует.
3. Представляет компактный отчёт, или экспортирует последние строки как CSV,
   когда аргумент — `csv`.

Используется `node`, а не `sqlite3`/`jq`, чтобы это работало идентично на
macOS, Linux и Windows.

## Отчёт

```bash
node -e '
const fs=require("fs"),os=require("os"),path=require("path");
const f=path.join(os.homedir(),".claude","metrics","costs.jsonl");
if(!fs.existsSync(f)){console.log("Cost tracker not set up: "+f+" not found. Enable the stop:cost-tracker hook and finish a session first.");process.exit(0);}
const rows=fs.readFileSync(f,"utf8").split(/\r?\n/).filter(Boolean).map(l=>{try{return JSON.parse(l)}catch{return null}}).filter(Boolean);
const bySession=new Map();
for(const r of rows){const k=r.session_id||r.transcript_path||r.timestamp;const p=bySession.get(k);if(!p||String(r.timestamp)>String(p.timestamp))bySession.set(k,r);}
const latest=[...bySession.values()];
const cost=r=>Number(r.estimated_cost_usd)||0;
const day=r=>String(r.timestamp||"").slice(0,10);
const today=new Date().toISOString().slice(0,10);
const d=new Date(Date.now()-864e5).toISOString().slice(0,10);
const sum=a=>a.reduce((s,r)=>s+cost(r),0);
const f4=n=>"$"+n.toFixed(4);
console.log("=== Cost summary ===");
console.log("today:     "+f4(sum(latest.filter(r=>day(r)===today))));
console.log("yesterday: "+f4(sum(latest.filter(r=>day(r)===d))));
console.log("total:     "+f4(sum(latest))+"  ("+latest.length+" sessions)");
const by=(key)=>{const m=new Map();for(const r of latest){const k=key(r)||"(unknown)";m.set(k,(m.get(k)||0)+cost(r));}return [...m.entries()].sort((a,b)=>b[1]-a[1]);};
console.log("\n=== By model ===");for(const [k,v] of by(r=>r.model))console.log(f4(v).padStart(12)+"  "+k);
console.log("\n=== Last 7 days ===");
const days=new Map();for(const r of latest){const k=day(r);days.set(k,(days.get(k)||0)+cost(r));}
[...days.entries()].sort((a,b)=>b[0]<a[0]?-1:1).slice(0,7).forEach(([k,v])=>console.log(k+"  "+f4(v)));
'
```

## Экспорт CSV (`/cost-report csv`)

```bash
node -e '
const fs=require("fs"),os=require("os"),path=require("path");
const f=path.join(os.homedir(),".claude","metrics","costs.jsonl");
if(!fs.existsSync(f)){console.error("no data");process.exit(0);}
const rows=fs.readFileSync(f,"utf8").split(/\r?\n/).filter(Boolean).map(l=>{try{return JSON.parse(l)}catch{return null}}).filter(Boolean).slice(-100);
console.log("timestamp,session_id,model,input_tokens,output_tokens,cache_write_tokens,cache_read_tokens,estimated_cost_usd");
for(const r of rows)console.log([r.timestamp,r.session_id,r.model,r.input_tokens,r.output_tokens,r.cache_write_tokens,r.cache_read_tokens,r.estimated_cost_usd].join(","));
'
```

## Формат отчёта

1. Резюме: сегодня, вчера, итого, количество сессий.
2. По моделям: модели, ранжированные по общей стоимости.
3. Последние семь дней: дата и стоимость.

Полагайтесь на предвычисленные значения `estimated_cost_usd`, записанные
трекером; не переоценивайте стоимость из сырых токенов здесь.
