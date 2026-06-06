import { useState, useRef, useEffect, useCallback } from "react";

// ─── SYSTEM PROMPT ───────────────────────────────────────────────────────────
const SENTINEL_SYSTEM_PROMPT = `You are SENTINEL — Abhisek's personal agentic AI. Elite operator. Three roles, one mission: grow Abhisek's passive income.

## IDENTITY
Name: SENTINEL | Owner: Abhisek | Mission: Build, handle, improve passive income — stocks, content, automation.

## ROLE 1: STOCK BROKER (PRIMARY)
- Deep knowledge: NSE/BSE, mutual funds, ETFs, SIPs, F&O basics, global macro.
- Always give: BUY / HOLD / WATCH / AVOID verdict + reasoning + Risk Level (Low/Medium/High).
- Auto-extract stock tickers from conversation. When you mention a stock, prefix it with $ (e.g. $TATAMOTORS, $HDFCBANK, $RELIANCE).
- NEVER execute large financial moves without "APPROVED" from Abhisek.
- Think like SEBI-registered advisor + hedge fund instincts.

## ROLE 2: PROGRAMMER
- Full-stack: Python, JS, React, Excel/VBA, APIs, automation, web scraping, AI integrations.
- Give working, production-ready code. No tutorials. Minimal explanations unless asked.

## ROLE 3: PERSONAL ASSISTANT + JUGAAD FINANCE ENGINE
### General Assistant:
- Tasks, summaries, planning, decisions, sounding board.
- Think ahead — anticipate what's needed next.

### Jugaad Finance Mode (YouTube channel for Indian audience, Hinglish style):
- SCRIPT WRITING: Full video scripts in conversational Hinglish. Hook → story → value → CTA structure.
- TITLE/HOOK GENERATOR: High-CTR YouTube titles. Fear + opportunity framing. Indian context.
- INSTAGRAM REELS: Short punchy scripts (30-60 sec), hook in first 3 words, strong visual cues.
- RESEARCH & FACT-CHECK: Verified stats from McKinsey, WEF, RBI, SEBI, AMFI. Always cite source + year.

## BEHAVIOR
- Tone: Optimistic, realistic, direct. No fluff.
- Format: Answer first. Reasoning second. Options third (if needed).
- Length: Short as possible. Long as necessary. Never padded.

## HARD RULES
1. Never generic advice — always specific to Abhisek's context.
2. Never execute big financial moves without explicit "APPROVED".
3. Never disclose personal data.
4. Never be a yes-man. Bad idea = say so + offer better alternative.
5. Always flag when data may be outdated (markets move fast).
6. When mentioning stocks, ALWAYS use $ prefix format: $SYMBOL.

## STOCK DETECTION
When you reference any stock/ticker in your response, format it as $SYMBOL so the UI can auto-detect and add to watchlist. Examples: $TATAMOTORS, $HDFCBANK, $NIFTY50, $RELIANCE, $INFY.

Respond as SENTINEL. Sharp. Useful. Execute.`;

// ─── CONSTANTS ────────────────────────────────────────────────────────────────
const MODES = {
  broker: { label: "BROKER", color: "#00ff88", icon: "📈" },
  dev: { label: "DEV", color: "#ff6b35", icon: "💻" },
  jugaad: { label: "JUGAAD", color: "#ffd700", icon: "🎬" },
  assistant: { label: "ASSIST", color: "#a78bfa", icon: "🧠" },
};

const STORAGE_KEY = "sentinel_memory_v1";
const WATCHLIST_KEY = "sentinel_watchlist_v1";

// ─── UTILS ───────────────────────────────────────────────────────────────────
const detectMode = (text) => {
  const t = text.toLowerCase();
  if (/stock|nse|bse|share|invest|buy|sell|portfolio|market|nifty|sensex|mutual fund|sip|etf|ticker|\$[a-z]/i.test(t)) return "broker";
  if (/code|script|function|python|javascript|react|bug|error|program|api|automat|def |import |const |class /i.test(t)) return "dev";
  if (/jugaad|youtube|reel|instagram|video|title|hook|script|hinglish|channel|content|subscriber/i.test(t)) return "jugaad";
  return "assistant";
};

const extractTickers = (text) => {
  const matches = text.match(/\$([A-Z][A-Z0-9]{1,14})/g) || [];
  return [...new Set(matches.map(m => m.slice(1)))];
};

const formatMsg = (text) =>
  text
    .replace(/\*\*(.*?)\*\*/g, "<strong>$1</strong>")
    .replace(/\*(.*?)\*/g, "<em>$1</em>")
    .replace(/`([^`]+)`/g, `<code style="background:#111;padding:2px 6px;border-radius:3px;font-family:'Courier New',monospace;color:#00d4ff;font-size:0.82em">$1</code>`)
    .replace(/\$([A-Z][A-Z0-9]{1,14})/g, `<span style="color:#00ff88;font-weight:700;background:rgba(0,255,136,0.08);padding:1px 5px;border-radius:3px;font-size:0.88em">$$$1</span>`)
    .replace(/\n/g, "<br/>");

const timestamp = () => new Date().toLocaleTimeString("en-IN", { hour: "2-digit", minute: "2-digit" });

const loadFromStorage = (key, fallback) => {
  try { return JSON.parse(localStorage.getItem(key)) || fallback; } catch { return fallback; }
};
const saveToStorage = (key, data) => {
  try { localStorage.setItem(key, JSON.stringify(data)); } catch {}
};

// ─── SUB-COMPONENTS ──────────────────────────────────────────────────────────
const Dot = ({ color, pulse }) => (
  <span style={{
    display: "inline-block", width: 7, height: 7, borderRadius: "50%",
    background: color, boxShadow: `0 0 6px ${color}`,
    animation: pulse ? "dotPulse 2s ease-in-out infinite" : "none",
    flexShrink: 0
  }} />
);

const Tag = ({ label, color }) => (
  <span style={{
    fontSize: "0.58em", fontWeight: 800, letterSpacing: "0.12em",
    padding: "2px 6px", borderRadius: 3, border: `1px solid ${color}44`,
    color, background: `${color}11`, textTransform: "uppercase"
  }}>{label}</span>
);

const TypingDots = () => (
  <div style={{ display: "flex", gap: 5, padding: "14px 16px", alignItems: "center" }}>
    {[0, 1, 2].map(i => (
      <div key={i} style={{
        width: 6, height: 6, borderRadius: "50%", background: "#00d4ff",
        animation: `dotPulse 1.2s ease-in-out ${i * 0.2}s infinite`
      }} />
    ))}
  </div>
);

// ─── WATCHLIST PANEL ─────────────────────────────────────────────────────────
const WatchlistPanel = ({ watchlist, onRemove }) => (
  <div style={{ padding: "12px 0" }}>
    <div style={{ fontSize: "0.6em", color: "#444", letterSpacing: "0.18em", padding: "0 16px 10px", borderBottom: "1px solid #111" }}>
      AUTO-DETECTED WATCHLIST
    </div>
    {watchlist.length === 0 ? (
      <div style={{ padding: "20px 16px", fontSize: "0.72em", color: "#333", textAlign: "center" }}>
        Sentinel will auto-add stocks<br />as you discuss them
      </div>
    ) : (
      watchlist.map((item, i) => (
        <div key={i} style={{
          display: "flex", alignItems: "center", justifyContent: "space-between",
          padding: "8px 16px", borderBottom: "1px solid #0d0d0d",
          transition: "background 0.15s"
        }}
          onMouseEnter={e => e.currentTarget.style.background = "#0a0a0a"}
          onMouseLeave={e => e.currentTarget.style.background = "transparent"}
        >
          <div>
            <div style={{ fontSize: "0.78em", color: "#00ff88", fontWeight: 700, letterSpacing: "0.08em" }}>
              ${item.symbol}
            </div>
            <div style={{ fontSize: "0.58em", color: "#444", marginTop: 2 }}>{item.addedAt}</div>
          </div>
          <button onClick={() => onRemove(i)} style={{
            background: "none", border: "none", color: "#333", cursor: "pointer",
            fontSize: "0.8em", padding: "2px 6px"
          }} onMouseEnter={e => e.target.style.color = "#ff4444"}
            onMouseLeave={e => e.target.style.color = "#333"}>✕</button>
        </div>
      ))
    )}
  </div>
< truncated lines 155-370 >
        <div style={{ display: "flex", alignItems: "center", gap: 12 }}>
          <div style={{ width: 36, height: 36, border: "1.5px solid #00d4ff", borderRadius: 7, display: "flex", alignItems: "center", justifyContent: "center", fontSize: 16, background: "rgba(0,212,255,0.06)", flexShrink: 0 }}>⬡</div>
          <div>
            <div style={{ fontSize: "1em", fontWeight: 900, letterSpacing: "0.18em", color: "#00d4ff", animation: "headerGlow 4s ease-in-out infinite" }}>SENTINEL</div>
            <div style={{ fontSize: "0.55em", color: "#444", letterSpacing: "0.16em" }}>PERSONAL AGENTIC AI · MVP</div>
          </div>
        </div>
        <div style={{ display: "flex", gap: 16, alignItems: "center" }}>
          {Object.entries(MODES).map(([k, v]) => (
            <div key={k} style={{ display: "flex", alignItems: "center", gap: 5 }}>
              <Dot color={v.color} pulse={true} />
              <span style={{ fontSize: "0.55em", color: "#444", letterSpacing: "0.1em" }}>{v.label}</span>
            </div>
          ))}
          <button onClick={() => setSideOpen(p => !p)} style={{
            background: sideOpen ? "rgba(0,212,255,0.08)" : "transparent",
            border: "1px solid #1a1a2e", borderRadius: 4, color: "#555",
            padding: "4px 8px", cursor: "pointer", fontSize: "0.7em"
          }}>⊞</button>
        </div>
      </div>

      {/* ── BODY ── */}
      <div style={{ flex: 1, display: "flex", overflow: "hidden" }}>

        {/* ── CHAT ── */}
        <div style={{ flex: 1, display: "flex", flexDirection: "column", overflow: "hidden" }}>

          {/* Quick actions */}
          <div style={{ display: "flex", gap: 6, padding: "8px 16px", borderBottom: "1px solid #0d0d0d", flexWrap: "wrap", flexShrink: 0 }}>
            {quickActions.map((q, i) => (
              <button key={i} onClick={() => sendMessage(q.text)} style={{
                background: "transparent", border: "1px solid #151520", color: "#555",
                padding: "4px 10px", borderRadius: 3, fontSize: "0.65em", cursor: "pointer",
                letterSpacing: "0.04em", transition: "all 0.15s"
              }}
                onMouseEnter={e => { e.target.style.borderColor = "#00d4ff33"; e.target.style.color = "#00d4ff88"; }}
                onMouseLeave={e => { e.target.style.borderColor = "#151520"; e.target.style.color = "#555"; }}
              >{q.label}</button>
            ))}
          </div>

          {/* Messages */}
          <div style={{ flex: 1, overflowY: "auto", padding: "16px", display: "flex", flexDirection: "column", gap: 12 }}>
            {messages.map((msg, i) => {
              const isUser = msg.role === "user";
              const mc = modeColor(msg.mode);
              return (
                <div key={i} className="msg-enter" style={{ display: "flex", justifyContent: isUser ? "flex-end" : "flex-start" }}>
                  <div style={{
                    maxWidth: "80%",
                    background: isUser ? `${mc}09` : "rgba(255,255,255,0.025)",
                    border: `1px solid ${isUser ? mc + "22" : "#151520"}`,
                    borderRadius: isUser ? "10px 10px 2px 10px" : "10px 10px 10px 2px",
                    padding: "10px 14px", fontSize: "0.82em", lineHeight: 1.75
                  }}>
                    <div style={{ display: "flex", alignItems: "center", gap: 6, marginBottom: 7 }}>
                      <Tag label={isUser ? "YOU" : "SENTINEL"} color={mc} />
                      {msg.mode && <Tag label={MODES[msg.mode]?.label} color={mc} />}
                      <span style={{ fontSize: "0.75em", color: "#2a2a3a", marginLeft: "auto" }}>{msg.time}</span>
                    </div>
                    <div dangerouslySetInnerHTML={{ __html: formatMsg(msg.content) }} />
                    {msg.tickers?.length > 0 && (
                      <div style={{ marginTop: 8, fontSize: "0.65em", color: "#00ff8866" }}>
                        ↑ Added to watchlist: {msg.tickers.map(t => `$${t}`).join(", ")}
                      </div>
                    )}
                  </div>
                </div>
              );
            })}
            {loading && (
              <div style={{ display: "flex" }}>
                <div style={{ background: "rgba(255,255,255,0.025)", border: "1px solid #151520", borderRadius: "10px 10px 10px 2px" }}>
                  <div style={{ fontSize: "0.55em", color: "#333", padding: "7px 14px 0", letterSpacing: "0.15em" }}>SENTINEL · PROCESSING</div>
                  <TypingDots />
                </div>
              </div>
            )}
            <div ref={bottomRef} />
          </div>

          {/* Input */}
          <div style={{ borderTop: "1px solid #00d4ff14", padding: "12px 16px", background: "rgba(0,212,255,0.015)", flexShrink: 0 }}>
            <div style={{ display: "flex", gap: 10, alignItems: "flex-end", border: "1px solid #151520", borderRadius: 7, padding: "8px 14px", background: "#06060a", transition: "border-color 0.2s" }}
              onFocusCapture={e => e.currentTarget.style.borderColor = "#00d4ff33"}
              onBlurCapture={e => e.currentTarget.style.borderColor = "#151520"}>
              <textarea ref={taRef} value={input} onChange={e => setInput(e.target.value)} onKeyDown={handleKey}
                placeholder="Give Sentinel a task..." rows={1}
                style={{ flex: 1, background: "transparent", border: "none", color: "#ddd", fontSize: "0.82em", lineHeight: 1.6, maxHeight: 100, overflow: "auto" }} />
              <button onClick={() => sendMessage()} disabled={loading || !input.trim()} style={{
                background: input.trim() && !loading ? "#00d4ff" : "#0d0d12",
                border: "none", borderRadius: 5, padding: "7px 14px",
                color: input.trim() && !loading ? "#000" : "#2a2a3a",
                fontSize: "0.68em", fontWeight: 900, letterSpacing: "0.1em",
                cursor: input.trim() && !loading ? "pointer" : "not-allowed",
                transition: "all 0.2s", whiteSpace: "nowrap"
              }}>EXECUTE ↵</button>
            </div>
            <div style={{ fontSize: "0.55em", color: "#222", marginTop: 6, textAlign: "center", letterSpacing: "0.08em" }}>
              ENTER to send · SHIFT+ENTER new line · Major financial actions require your approval
            </div>
          </div>
        </div>

        {/* ── SIDE PANEL ── */}
        {sideOpen && (
          <div style={{ width: 230, borderLeft: "1px solid #0d0d12", display: "flex", flexDirection: "column", overflow: "hidden", flexShrink: 0 }}>
            {/* Panel tabs */}
            <div style={{ display: "flex", borderBottom: "1px solid #0d0d12", flexShrink: 0 }}>
              {[
                { id: "watchlist", icon: "📈", label: "Watch" },
                { id: "memory", icon: "🧠", label: "Memory" },
                { id: "jugaad", icon: "🎬", label: "Jugaad" },
              ].map(tab => (
                <button key={tab.id} onClick={() => setSidePanel(tab.id)} style={{
                  flex: 1, padding: "8px 4px", background: sidePanel === tab.id ? "rgba(0,212,255,0.05)" : "transparent",
                  border: "none", borderBottom: sidePanel === tab.id ? "1px solid #00d4ff44" : "1px solid transparent",
                  color: sidePanel === tab.id ? "#00d4ff" : "#444",
                  cursor: "pointer", fontSize: "0.6em", letterSpacing: "0.08em",
                  display: "flex", flexDirection: "column", alignItems: "center", gap: 2
                }}>
                  <span style={{ fontSize: "1.4em" }}>{tab.icon}</span>
                  {tab.label}
                </button>
              ))}
            </div>
            {/* Panel content */}
            <div style={{ flex: 1, overflowY: "auto" }}>
              {sidePanel === "watchlist" && (
                <WatchlistPanel
                  watchlist={watchlist}
                  onRemove={i => setWatchlist(prev => prev.filter((_, idx) => idx !== i))}
                />
              )}
              {sidePanel === "memory" && (
                <MemoryPanel
                  memory={memory}
                  onAdd={text => setMemory(prev => [{ text, savedAt: timestamp() }, ...prev])}
                  onDelete={i => setMemory(prev => prev.filter((_, idx) => idx !== i))}
                  onExport={exportMemory}
                />
              )}
              {sidePanel === "jugaad" && (
                <JugaadPanel onPrompt={text => { setInput(text); setSidePanel("watchlist"); taRef.current?.focus(); }} />
              )}
            </div>
          </div>
        )}
      </div>
    </div>
  );
}
