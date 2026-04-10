import { useState, useRef, useEffect } from "react";

const COLORS = {
  bg: "#0a0a0f", bg2: "#111118", bg3: "#1a1a24", border: "#2a2a3a",
  accent: "#6c63ff", accent2: "#ff6584", accent3: "#43e97b",
  text: "#e8e8f0", text2: "#9090a8", text3: "#5a5a72", card: "#14141e",
};

const s = {
  app: { display:"flex", flexDirection:"column", height:"100vh", background:COLORS.bg, color:COLORS.text, fontFamily:"system-ui,sans-serif", overflow:"hidden", position:"relative" },
  topbar: { display:"flex", alignItems:"center", justifyContent:"space-between", padding:"12px 20px", borderBottom:`1px solid ${COLORS.border}`, background:"rgba(10,10,15,0.95)", flexShrink:0 },
  logo: { fontSize:18, fontWeight:800, display:"flex", alignItems:"center", gap:8 },
  logoDot: { width:8, height:8, borderRadius:"50%", background:COLORS.accent, boxShadow:`0 0 10px ${COLORS.accent}` },
  badge: { fontFamily:"monospace", fontSize:10, padding:"4px 10px", borderRadius:6, background:COLORS.bg3, border:`1px solid ${COLORS.border}`, color:COLORS.accent3 },
  nav: { display:"flex", gap:4 },
  navBtn: (active) => ({ padding:"6px 14px", borderRadius:8, border: active?`1px solid ${COLORS.border}`:"none", background: active?COLORS.bg3:"transparent", color: active?COLORS.text:COLORS.text2, fontFamily:"inherit", fontSize:12, cursor:"pointer" }),
  main: { flex:1, display:"grid", gridTemplateColumns:"240px 1fr 280px", overflow:"hidden" },
  sidebar: { borderRight:`1px solid ${COLORS.border}`, padding:"16px 12px", display:"flex", flexDirection:"column", gap:6, overflowY:"auto", background:"rgba(10,10,15,0.5)" },
  sidebarSection: { fontFamily:"monospace", fontSize:10, color:COLORS.text3, letterSpacing:2, padding:"6px 10px", textTransform:"uppercase" },
  sidebarItem: (active) => ({ display:"flex", alignItems:"center", gap:10, padding:"9px 10px", borderRadius:10, cursor:"pointer", border: active?`1px solid ${COLORS.accent}`:"1px solid transparent", background: active?`rgba(108,99,255,0.15)`:"transparent", color: active?COLORS.accent:COLORS.text }),
  profileCard: { borderRadius:14, padding:14, background:`linear-gradient(135deg,${COLORS.bg3},${COLORS.bg2})`, border:`1px solid ${COLORS.border}`, marginBottom:8 },
  profileAvatar: { width:40, height:40, borderRadius:10, background:`linear-gradient(135deg,${COLORS.accent},${COLORS.accent2})`, display:"flex", alignItems:"center", justifyContent:"center", fontSize:18, marginBottom:8 },
  center: { display:"flex", flexDirection:"column", overflow:"hidden" },
  panelHeader: { padding:"16px 22px", borderBottom:`1px solid ${COLORS.border}`, display:"flex", alignItems:"center", justifyContent:"space-between", flexShrink:0 },
  panelTitle: { fontSize:20, fontWeight:800 },
  panelContent: { flex:1, overflowY:"auto", padding:"18px 22px" },
  card: { background:COLORS.card, border:`1px solid ${COLORS.border}`, borderRadius:14, padding:16, marginBottom:14 },
  cardTitle: { fontFamily:"monospace", fontSize:10, fontWeight:700, color:COLORS.text2, letterSpacing:1, textTransform:"uppercase", marginBottom:12 },
  uploadZone: (drag) => ({ border:`2px dashed ${drag?COLORS.accent:COLORS.border}`, borderRadius:14, padding:40, textAlign:"center", cursor:"pointer", background: drag?`rgba(108,99,255,0.08)`:"transparent", transition:"all .3s" }),
  btn: (variant) => ({ padding: variant==="sm"?"6px 14px":"9px 18px", borderRadius:8, border:"none", background: variant==="secondary"?COLORS.bg3:COLORS.accent, color: variant==="secondary"?COLORS.text2:"white", fontFamily:"inherit", fontSize:12, fontWeight:700, cursor:"pointer" }),
  rightPanel: { borderLeft:`1px solid ${COLORS.border}`, display:"flex", flexDirection:"column", overflow:"hidden", background:"rgba(10,10,15,0.3)" },
  rpHeader: { padding:16, borderBottom:`1px solid ${COLORS.border}`, fontSize:13, fontWeight:700 },
  rpContent: { flex:1, overflowY:"auto", padding:12 },
  alert: (type) => ({ padding:10, borderRadius:10, marginBottom:8, display:"flex", gap:10, alignItems:"flex-start", background: type==="danger"?"rgba(255,101,132,0.1)":type==="warning"?"rgba(255,215,0,0.08)":"rgba(67,233,123,0.08)", border: type==="danger"?"1px solid rgba(255,101,132,0.3)":type==="warning"?"1px solid rgba(255,215,0,0.2)":"1px solid rgba(67,233,123,0.2)" }),
  companyItem: { display:"flex", alignItems:"center", gap:10, padding:10, borderRadius:9, background:COLORS.bg2, border:`1px solid ${COLORS.border}`, marginBottom:7 },
  msgBubble: (isUser) => ({ maxWidth:"78%", padding:"10px 14px", borderRadius:14, fontSize:13, lineHeight:1.6, background: isUser?COLORS.accent:COLORS.bg3, color: isUser?"white":COLORS.text, border: isUser?"none":`1px solid ${COLORS.border}`, borderBottomRightRadius: isUser?4:14, borderBottomLeftRadius: isUser?14:4 }),
  chatInput: { flex:1, background:COLORS.bg3, border:`1px solid ${COLORS.border}`, borderRadius:10, padding:"10px 14px", color:COLORS.text, fontFamily:"inherit", fontSize:13, outline:"none", resize:"none", minHeight:42, maxHeight:100 },
  sendBtn: (disabled) => ({ width:42, height:42, borderRadius:10, background: disabled?"#333":COLORS.accent, border:"none", color:"white", fontSize:17, cursor: disabled?"not-allowed":"pointer", display:"flex", alignItems:"center", justifyContent:"center", opacity: disabled?0.4:1 }),
};

async function callClaude(messages, maxTokens = 1000) {
  let system = "";
  const userMessages = [];
  for (const m of messages) {
    if (m.role === "system") system = m.content;
    else userMessages.push(m);
  }
  const body = { model: "claude-sonnet-4-20250514", max_tokens: maxTokens, messages: userMessages };
  if (system) body.system = system;
  const res = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
  });
  const data = await res.json();
  if (data.error) throw new Error(data.error.message);
  return data.content.map(b => b.text || "").join("");
}

async function extractTextFromPDF(file) {
  return new Promise((resolve) => {
    const reader = new FileReader();
    reader.onload = async (e) => {
      try {
        if (!window.pdfjsLib) {
          await new Promise((res, rej) => {
            const sc = document.createElement("script");
            sc.src = "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.min.js";
            sc.onload = res; sc.onerror = rej;
            document.head.appendChild(sc);
          });
          window.pdfjsLib.GlobalWorkerOptions.workerSrc =
            "https://cdnjs.cloudflare.com/ajax/libs/pdf.js/3.11.174/pdf.worker.min.js";
        }
        const pdf = await window.pdfjsLib.getDocument({ data: new Uint8Array(e.target.result) }).promise;
        let text = "";
        for (let i = 1; i <= pdf.numPages; i++) {
          const page = await pdf.getPage(i);
          const content = await page.getTextContent();
          text += content.items.map(it => it.str).join(" ") + "\n";
        }
        text = text.replace(/\s{3,}/g, "\n").trim();
        resolve(text.length > 80 ? text : `Resume: ${file.name}`);
      } catch { resolve(`Resume file: ${file.name}`); }
    };
    reader.onerror = () => resolve(`Resume: ${file.name}`);
    reader.readAsArrayBuffer(file);
  });
}

const DEFAULT_PROFILE = (name) => ({
  name, branch: "Computer Science Engineering", year: "2025",
  skills: [
    { name: "Data Structures", level: 55, category: "DSA" },
    { name: "Algorithms", level: 45, category: "DSA" },
    { name: "Python", level: 65, category: "Dev" },
    { name: "SQL", level: 50, category: "Core" },
  ],
  targetRoles: ["SDE", "Full Stack Developer"],
  targetCompanies: [
    { name: "TCS", logo: "T", color: "#0056a2", role: "Developer", matchScore: 80, tier: "safe" },
    { name: "Infosys", logo: "I", color: "#7b2d8b", role: "Systems Engineer", matchScore: 72, tier: "safe" },
    { name: "Google", logo: "G", color: "#4285f4", role: "SWE", matchScore: 25, tier: "dream" },
  ],
  gaps: ["DSA needs improvement", "No system design experience", "Missing GitHub projects"],
  strengths: ["Good communication", "Basic programming knowledge"],
  readinessScore: 55,
  weeklyPlan: [
    { week: 1, title: "DSA Foundations", focus: "Arrays & Strings", tasks: ["Solve 20 LeetCode easy", "Study sorting", "Practice binary search"], status: "current" },
    { week: 2, title: "Core DSA", focus: "Trees & Graphs", tasks: ["Tree traversals", "Graph BFS/DFS", "10 medium problems"], status: "upcoming" },
    { week: 3, title: "Projects", focus: "Portfolio", tasks: ["Build full-stack project", "Push to GitHub", "Update LinkedIn"], status: "upcoming" },
    { week: 4, title: "Mock & Apply", focus: "Interview Prep", tasks: ["5 mock interviews", "Apply to 10 companies", "HR preparation"], status: "upcoming" },
  ],
  alerts: [
    { type: "danger", title: "DSA Gap", message: "DSA score below threshold for product companies." },
    { type: "warning", title: "No GitHub Activity", message: "Add 2-3 projects for recruiters." },
    { type: "success", title: "Communication Strong", message: "Soft skills are interview-ready." },
  ],
  agenticDecisions: [{ action: "Assigned 4-week DSA practice plan" }],
});

export default function PlaceAI() {
  const [view, setView] = useState("dashboard");
  const [profile, setProfile] = useState(null);
  const [loading, setLoading] = useState(false);
  const [loadingText, setLoadingText] = useState("Analyzing resume...");
  const [drag, setDrag] = useState(false);
  const [messages, setMessages] = useState([{ role: "ai", text: "Hi! I'm your AI Career Coach. Upload your resume on the Dashboard to get a personalized analysis!" }]);
  const [chatInput, setChatInput] = useState("");
  const [chatLoading, setChatLoading] = useState(false);
  const [chatHistory, setChatHistory] = useState([]);
  const [interviewDomain, setInterviewDomain] = useState("Software Engineering");
  const [interviewStarted, setInterviewStarted] = useState(false);
  const [interviewQuestion, setInterviewQuestion] = useState("");
  const [interviewAnswer, setInterviewAnswer] = useState("");
  const [interviewFeedback, setInterviewFeedback] = useState("");
  const [interviewScores, setInterviewScores] = useState([]);
  const [questionNum, setQuestionNum] = useState(1);
  const [interviewDone, setInterviewDone] = useState(false);
  const [interviewLoading, setInterviewLoading] = useState(false);
  const fileRef = useRef();
  const chatEndRef = useRef();
  const chatInputRef = useRef();

  useEffect(() => { chatEndRef.current?.scrollIntoView({ behavior: "smooth" }); }, [messages]);

  async function processFile(file) {
    setLoading(true);
    setLoadingText("Reading your resume...");
    let text = file.type === "application/pdf" ? await extractTextFromPDF(file) : await file.text();
    setLoadingText("AI is analyzing your profile...");
    const prompt = `You are an expert placement counselor for Indian campus placements. Analyze this resume carefully and return ONLY a pure JSON object (no markdown, no backticks, no extra text).

Extract ALL real details: actual student name, actual branch/degree, actual skills, actual projects, actual CGPA if present. Do NOT use placeholder data.

JSON structure:
{"name":"ACTUAL name from resume","branch":"ACTUAL branch","year":"graduation year","skills":[{"name":"skill","level":70,"category":"DSA or Dev or Core or Soft"}],"targetRoles":["role1"],"targetCompanies":[{"name":"Company","logo":"C","color":"#hex","role":"Role","matchScore":80,"tier":"safe or moderate or dream"}],"gaps":["specific gap"],"strengths":["specific strength"],"readinessScore":65,"weeklyPlan":[{"week":1,"title":"Title","focus":"topic","tasks":["task1","task2","task3"],"status":"current"},{"week":2,"title":"Title","focus":"topic","tasks":["task1","task2","task3"],"status":"upcoming"},{"week":3,"title":"Title","focus":"topic","tasks":["task1","task2","task3"],"status":"upcoming"},{"week":4,"title":"Title","focus":"topic","tasks":["task1","task2","task3"],"status":"upcoming"}],"alerts":[{"type":"danger or warning or success","title":"title","message":"specific message"}],"agenticDecisions":[{"action":"specific action for this student"}]}

Resume text:
${text.substring(0, 3000)}

File: ${file.name}

IMPORTANT: Use REAL data from the resume above. Every field must reflect actual content.`;
    try {
      const res = await callClaude([{ role: "user", content: prompt }], 1800);
      const clean = res.replace(/```json|```/g, "").trim();
      const start = clean.indexOf("{"), end = clean.lastIndexOf("}");
      const parsed = JSON.parse(clean.substring(start, end + 1));
      setProfile(parsed);
      setMessages(prev => [...prev, { role: "ai", text: `Profile loaded! I've analyzed **${parsed.name}**'s resume.\n\n**Readiness: ${parsed.readinessScore}%**\n\nKey gaps: ${parsed.gaps.slice(0,2).join(", ")}\n\nAsk me anything about your placement preparation!` }]);
    } catch {
      const fallback = DEFAULT_PROFILE(file.name.replace(/\.[^/.]+$/, "").replace(/[-_]/g, " ") || "Student");
      setProfile(fallback);
      setMessages(prev => [...prev, { role: "ai", text: `Resume uploaded! Here's a general profile. For best results, make sure your resume has clear text content.` }]);
    }
    setLoading(false);
  }

  async function sendChat() {
    if (!chatInput.trim() || chatLoading) return;
    const text = chatInput.trim();
    setChatInput("");
    setMessages(prev => [...prev, { role: "user", text }]);
    setChatLoading(true);
    const sys = `You are an Agentic AI Career Coach for Indian university placements. Be direct, mentor-like, specific.
${profile ? `Student: ${profile.name}, ${profile.branch}, Readiness: ${profile.readinessScore}%, Gaps: ${profile.gaps.join(", ")}, Strengths: ${profile.strengths.join(", ")}` : "No resume uploaded yet."}
Keep response under 150 words unless technical. Use **bold** for key points.`;
    const newHistory = [...chatHistory, { role: "user", content: text }];
    setChatHistory(newHistory);
    try {
      const reply = await callClaude([{ role: "system", content: sys }, ...newHistory.slice(-8)], 400);
      setChatHistory(h => [...h, { role: "assistant", content: reply }]);
      setMessages(prev => [...prev, { role: "ai", text: reply }]);
    } catch (e) {
      setMessages(prev => [...prev, { role: "ai", text: "Error: " + e.message }]);
    }
    setChatLoading(false);
  }

  async function startInterview() {
    setInterviewStarted(true);
    setInterviewDone(false);
    setInterviewScores([]);
    setQuestionNum(1);
    setInterviewFeedback("");
    setInterviewAnswer("");
    await loadQuestion(interviewDomain);
  }

  async function loadQuestion(domain) {
    setInterviewLoading(true);
    setInterviewQuestion("");
    setInterviewFeedback("");
    setInterviewAnswer("");
    try {
      const q = await callClaude([{ role: "user", content: `Generate one ${domain} interview question for a fresh CS graduate. Return ONLY the question, nothing else.` }], 100);
      setInterviewQuestion(q);
    } catch {
      const fallbacks = { "Software Engineering": "Explain REST vs GraphQL. When would you use each?", "DSA": "Given an array, find two numbers summing to a target. Explain time complexity.", "System Design": "Design a URL shortening service. What components would you include?", "HR Round": "Tell me about a time you faced conflict in a team. How did you resolve it?", "Data Science": "Explain overfitting vs underfitting. How do you handle each?" };
      setInterviewQuestion(fallbacks[domain] || fallbacks["Software Engineering"]);
    }
    setInterviewLoading(false);
  }

  async function submitAnswer() {
    if (!interviewAnswer.trim()) return;
    setInterviewLoading(true);
    try {
      const fb = await callClaude([{ role: "user", content: `Evaluate this interview answer briefly.\nQuestion: ${interviewQuestion}\nAnswer: ${interviewAnswer}\nGive: score (X/10), one strength, one improvement. Under 60 words. Format: "Score: X/10 | Strength: ... | Improve: ..."` }], 120);
      setInterviewFeedback(fb);
      const match = fb.match(/Score:\s*(\d+)/i);
      if (match) setInterviewScores(prev => [...prev, parseInt(match[1])]);
    } catch { setInterviewFeedback("Could not evaluate. Try again."); }
    setInterviewLoading(false);
  }

  async function nextQuestion() {
    if (questionNum >= 5) { setInterviewDone(true); return; }
    setQuestionNum(n => n + 1);
    await loadQuestion(interviewDomain);
  }

  const avg = interviewScores.length ? (interviewScores.reduce((a,b)=>a+b,0)/interviewScores.length).toFixed(1) : null;

  return (
    <div style={s.app}>
      {/* TOPBAR */}
      <div style={s.topbar}>
        <div style={s.logo}><div style={s.logoDot}/> PlaceAI</div>
        <div style={s.nav}>
          {["dashboard","coach","interview","roadmap"].map(v => (
            <button key={v} style={s.navBtn(view===v)} onClick={() => setView(v)}>
              {v.charAt(0).toUpperCase()+v.slice(1)}
            </button>
          ))}
        </div>
        <div style={s.badge}>⬤ AI POWERED · FREE</div>
      </div>

      <div style={s.main}>
        {/* SIDEBAR */}
        <div style={s.sidebar}>
          <div style={s.profileCard}>
            <div style={s.profileAvatar}>👤</div>
            <div style={{ fontSize:14, fontWeight:700, marginBottom:2 }}>{profile?.name || "Upload Resume"}</div>
            <div style={{ fontSize:11, color:COLORS.text2, fontFamily:"monospace" }}>{profile ? `${profile.branch} · ${profile.year}` : "to get started"}</div>
            {profile && (
              <div style={{ marginTop:12 }}>
                <div style={{ display:"flex", justifyContent:"space-between", fontSize:11, color:COLORS.text2, marginBottom:5, fontFamily:"monospace" }}>
                  <span>Placement Readiness</span><span>{profile.readinessScore}%</span>
                </div>
                <div style={{ height:4, background:COLORS.border, borderRadius:99, overflow:"hidden" }}>
                  <div style={{ height:"100%", width:`${profile.readinessScore}%`, background:`linear-gradient(90deg,${COLORS.accent},${COLORS.accent3})`, borderRadius:99, transition:"width 1s" }}/>
                </div>
              </div>
            )}
          </div>
          <div style={s.sidebarSection}>Navigation</div>
          {[["dashboard","📊","Dashboard"],["coach","🤖","AI Coach"],["interview","🎤","Mock Interview"],["roadmap","🗺️","Roadmap"]].map(([v,icon,label]) => (
            <div key={v} style={s.sidebarItem(view===v)} onClick={() => setView(v)}>
              <span style={{ fontSize:14 }}>{icon}</span>
              <span style={{ fontSize:13, fontWeight:600 }}>{label}</span>
            </div>
          ))}
          {profile && (
            <>
              <div style={s.sidebarSection}>Agentic Actions</div>
              {profile.agenticDecisions.map((d, i) => (
                <div key={i} style={{ padding:"7px 10px", borderRadius:7, background:"rgba(108,99,255,0.08)", border:"1px solid rgba(108,99,255,0.2)", marginBottom:4 }}>
                  <div style={{ fontFamily:"monospace", fontSize:9, color:COLORS.accent, marginBottom:2 }}>AUTO-ACTION</div>
                  <div style={{ fontSize:11, lineHeight:1.4 }}>{d.action}</div>
                </div>
              ))}
            </>
          )}
        </div>

        {/* CENTER */}
        <div style={s.center}>
          {/* DASHBOARD */}
          {view === "dashboard" && (
            <>
              <div style={s.panelHeader}>
                <div style={s.panelTitle}>Placement <span style={{ color:COLORS.accent }}>Intelligence</span></div>
              </div>
              <div style={s.panelContent}>
                {!profile ? (
                  <div
                    style={s.uploadZone(drag)}
                    onDragOver={e => { e.preventDefault(); setDrag(true); }}
                    onDragLeave={() => setDrag(false)}
                    onDrop={e => { e.preventDefault(); setDrag(false); if (e.dataTransfer.files[0]) processFile(e.dataTransfer.files[0]); }}
                    onClick={() => fileRef.current.click()}
                  >
                    <div style={{ fontSize:40, marginBottom:14 }}>📄</div>
                    <div style={{ fontSize:17, fontWeight:700, marginBottom:6 }}>Drop your resume here</div>
                    <div style={{ fontSize:12, color:COLORS.text2, fontFamily:"monospace" }}>PDF or TXT · AI analyzes instantly</div>
                    <button style={{ ...s.btn(), marginTop:18 }} onClick={e => { e.stopPropagation(); fileRef.current.click(); }}>Choose File</button>
                    <input ref={fileRef} type="file" accept=".pdf,.txt" style={{ display:"none" }} onChange={e => { if (e.target.files[0]) processFile(e.target.files[0]); }}/>
                  </div>
                ) : (
                  <>
                    <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr 1fr", gap:10, marginBottom:16 }}>
                      {[
                        { label:"Readiness", val:`${profile.readinessScore}%`, color:COLORS.accent },
                        { label:"Good Matches", val:profile.targetCompanies.filter(c=>c.matchScore>70).length, color:COLORS.accent3 },
                        { label:"Skill Gaps", val:profile.gaps.length, color:COLORS.accent2 },
                      ].map(stat => (
                        <div key={stat.label} style={{ padding:14, borderRadius:10, background:COLORS.bg2, border:`1px solid ${COLORS.border}`, textAlign:"center" }}>
                          <div style={{ fontSize:26, fontWeight:800, color:stat.color }}>{stat.val}</div>
                          <div style={{ fontFamily:"monospace", fontSize:10, color:COLORS.text3, marginTop:3, textTransform:"uppercase" }}>{stat.label}</div>
                        </div>
                      ))}
                    </div>
                    <div style={s.card}>
                      <div style={s.cardTitle}>Skill Analysis</div>
                      <div style={{ display:"grid", gridTemplateColumns:"1fr 1fr", gap:8 }}>
                        {profile.skills.map((sk, i) => (
                          <div key={i} style={{ padding:12, borderRadius:10, border:`1px solid ${COLORS.border}`, background:COLORS.bg2 }}>
                            <div style={{ fontSize:12, fontWeight:600, marginBottom:5 }}>{sk.name}</div>
                            <div style={{ display:"flex", alignItems:"center", gap:8 }}>
                              <div style={{ flex:1, height:3, background:COLORS.border, borderRadius:99, overflow:"hidden" }}>
                                <div style={{ height:"100%", width:`${sk.level}%`, borderRadius:99, background: sk.level>=70?COLORS.accent3:sk.level>=50?"#ffd700":COLORS.accent2 }}/>
                              </div>
                              <span style={{ fontFamily:"monospace", fontSize:11, color:COLORS.text2 }}>{sk.level}%</span>
                            </div>
                          </div>
                        ))}
                      </div>
                    </div>
                    <div style={s.card}>
                      <div style={s.cardTitle}>Strengths & Gaps</div>
                      <div style={{ marginBottom:8 }}>
                        {profile.strengths.map((st,i) => <div key={i} style={{ fontSize:12, color:COLORS.accent3, marginBottom:4 }}>✅ {st}</div>)}
                      </div>
                      {profile.gaps.map((g,i) => <div key={i} style={{ fontSize:12, color:COLORS.accent2, marginBottom:4 }}>⚠️ {g}</div>)}
                    </div>
                    <button style={{ ...s.btn("secondary"), fontSize:11, marginTop:4 }} onClick={() => { setProfile(null); fileRef.current && (fileRef.current.value=""); }}>
                      Upload Different Resume
                    </button>
                  </>
                )}
              </div>
            </>
          )}

          {/* COACH */}
          {view === "coach" && (
            <div style={{ display:"flex", flexDirection:"column", height:"100%" }}>
              <div style={s.panelHeader}>
                <div style={s.panelTitle}>Agentic <span style={{ color:COLORS.accent }}>Career Coach</span></div>
                <div style={{ fontSize:11, color:COLORS.text2, fontFamily:"monospace" }}>Powered by Claude AI</div>
              </div>
              <div style={{ flex:1, overflowY:"auto", padding:16, display:"flex", flexDirection:"column", gap:12 }}>
                {messages.map((m, i) => (
                  <div key={i} style={{ display:"flex", gap:10, flexDirection: m.role==="user"?"row-reverse":"row" }}>
                    <div style={{ width:30, height:30, borderRadius:8, background: m.role==="user"?`linear-gradient(135deg,${COLORS.accent2},#ff8fab)`:`linear-gradient(135deg,${COLORS.accent},#9c88ff)`, display:"flex", alignItems:"center", justifyContent:"center", fontSize:13, flexShrink:0 }}>
                      {m.role==="user"?"👤":"🤖"}
                    </div>
                    <div style={s.msgBubble(m.role==="user")} dangerouslySetInnerHTML={{ __html: m.text.replace(/\*\*(.*?)\*\*/g,"<strong>$1</strong>").replace(/\n/g,"<br/>") }}/>
                  </div>
                ))}
                {chatLoading && (
                  <div style={{ display:"flex", gap:10 }}>
                    <div style={{ width:30, height:30, borderRadius:8, background:`linear-gradient(135deg,${COLORS.accent},#9c88ff)`, display:"flex", alignItems:"center", justifyContent:"center", fontSize:13 }}>🤖</div>
                    <div style={{ padding:"10px 14px", borderRadius:14, background:COLORS.bg3, border:`1px solid ${COLORS.border}` }}>
                      <div style={{ display:"flex", gap:4 }}>{[0,1,2].map(i => <div key={i} style={{ width:5, height:5, borderRadius:"50%", background:COLORS.accent, animation:`bounce 0.8s ${i*0.15}s infinite` }}/>)}</div>
                    </div>
                  </div>
                )}
                <div ref={chatEndRef}/>
              </div>
              <div style={{ padding:12, borderTop:`1px solid ${COLORS.border}`, flexShrink:0 }}>
                <div style={{ display:"flex", gap:8, alignItems:"flex-end" }}>
                  <textarea
                    ref={chatInputRef}
                    style={s.chatInput}
                    placeholder="Ask your coach anything..."
                    value={chatInput}
                    onChange={e => setChatInput(e.target.value)}
                    onKeyDown={e => { if (e.key==="Enter" && !e.shiftKey) { e.preventDefault(); sendChat(); } }}
                    rows={1}
                  />
                  <button style={s.sendBtn(chatLoading)} disabled={chatLoading} onClick={sendChat}>↑</button>
                </div>
                <div style={{ fontSize:10, color:COLORS.text3, marginTop:4, fontFamily:"monospace" }}>ENTER to send · SHIFT+ENTER for new line</div>
              </div>
            </div>
          )}

          {/* INTERVIEW */}
          {view === "interview" && (
            <>
              <div style={s.panelHeader}>
                <div style={s.panelTitle}>Mock <span style={{ color:COLORS.accent }}>Interview</span></div>
                <div style={{ display:"flex", gap:8, alignItems:"center" }}>
                  <select value={interviewDomain} onChange={e => setInterviewDomain(e.target.value)} style={{ background:COLORS.bg3, border:`1px solid ${COLORS.border}`, borderRadius:8, padding:"6px 10px", color:COLORS.text, fontFamily:"inherit", fontSize:12, outline:"none" }}>
                    {["Software Engineering","Data Science","System Design","HR Round","DSA"].map(d => <option key={d}>{d}</option>)}
                  </select>
                  <button style={s.btn()} onClick={startInterview}>Start Session</button>
                </div>
              </div>
              <div style={s.panelContent}>
                {!interviewStarted ? (
                  <div style={{ textAlign:"center", padding:40 }}>
                    <div style={{ fontSize:42, marginBottom:16 }}>🎤</div>
                    <div style={{ fontSize:18, fontWeight:700, marginBottom:6 }}>Ready for your Mock Interview?</div>
                    <div style={{ color:COLORS.text2, fontSize:12, fontFamily:"monospace" }}>AI asks questions · Evaluates answers · Real feedback</div>
                    <button style={{ ...s.btn(), marginTop:20 }} onClick={startInterview}>Begin Interview →</button>
                  </div>
                ) : interviewDone ? (
                  <div style={{ textAlign:"center", padding:40, background:COLORS.card, borderRadius:14, border:`1px solid ${COLORS.border}` }}>
                    <div style={{ fontSize:42, marginBottom:14 }}>🏆</div>
                    <div style={{ fontSize:20, fontWeight:800, marginBottom:6 }}>Session Complete!</div>
                    <div style={{ fontSize:32, fontWeight:800, color:COLORS.accent, margin:"14px 0" }}>{avg}/10</div>
                    <div style={{ color:COLORS.text2, fontSize:12, fontFamily:"monospace" }}>Average across {interviewScores.length} questions</div>
                    <button style={{ ...s.btn(), marginTop:20 }} onClick={startInterview}>New Session</button>
                  </div>
                ) : (
                  <>
                    <div style={{ display:"flex", justifyContent:"space-between", marginBottom:14, fontFamily:"monospace", fontSize:12, color:COLORS.text2 }}>
                      <span>Question {questionNum} of 5</span>
                      {avg && <span style={{ color:COLORS.accent3 }}>Avg: {avg}/10</span>}
                    </div>
                    <div style={{ background:`linear-gradient(135deg,rgba(108,99,255,0.1),rgba(156,136,255,0.05))`, border:`1px solid rgba(108,99,255,0.3)`, borderRadius:14, padding:18, marginBottom:14 }}>
                      <div style={{ fontFamily:"monospace", fontSize:10, color:COLORS.accent, marginBottom:6, letterSpacing:2 }}>INTERVIEWER</div>
                      <div style={{ fontSize:15, fontWeight:600, lineHeight:1.5, marginBottom:14, fontStyle:"italic" }}>
                        {interviewLoading ? "Loading question..." : interviewQuestion}
                      </div>
                      {interviewFeedback && (
                        <div style={{ padding:10, background:"rgba(67,233,123,0.08)", border:"1px solid rgba(67,233,123,0.2)", borderRadius:9, fontSize:12, lineHeight:1.6 }}>
                          ✅ <strong>Feedback:</strong> {interviewFeedback}
                        </div>
                      )}
                    </div>
                    {!interviewFeedback ? (
                      <>
                        <textarea
                          value={interviewAnswer}
                          onChange={e => setInterviewAnswer(e.target.value)}
                          placeholder="Type your answer here..."
                          style={{ width:"100%", background:COLORS.bg2, border:`1px solid ${COLORS.border}`, borderRadius:9, padding:10, color:COLORS.text, fontFamily:"inherit", fontSize:13, resize:"none", outline:"none", minHeight:75, boxSizing:"border-box" }}
                        />
                        <div style={{ display:"flex", gap:8, marginTop:10 }}>
                          <button style={s.btn()} onClick={submitAnswer} disabled={interviewLoading}>{interviewLoading?"Evaluating...":"Submit Answer"}</button>
                          <button style={s.btn("secondary")} onClick={nextQuestion}>Skip →</button>
                        </div>
                      </>
                    ) : (
                      <button style={s.btn()} onClick={nextQuestion}>{questionNum >= 5 ? "See Results" : "Next Question →"}</button>
                    )}
                  </>
                )}
              </div>
            </>
          )}

          {/* ROADMAP */}
          {view === "roadmap" && (
            <>
              <div style={s.panelHeader}>
                <div style={s.panelTitle}>Prep <span style={{ color:COLORS.accent }}>Roadmap</span></div>
                <div style={{ fontSize:11, color:COLORS.text2, fontFamily:"monospace" }}>AI-generated · Personalized</div>
              </div>
              <div style={s.panelContent}>
                {!profile ? (
                  <div style={{ textAlign:"center", padding:50, color:COLORS.text2 }}>
                    <div style={{ fontSize:36, marginBottom:14 }}>🗺️</div>
                    <div style={{ fontWeight:700, marginBottom:6 }}>No roadmap yet</div>
                    <div style={{ fontSize:12, fontFamily:"monospace" }}>Upload your resume to generate a personalized plan</div>
                  </div>
                ) : profile.weeklyPlan.map((w, i) => (
                  <div key={i} style={{ marginBottom:18, position:"relative", paddingLeft:22 }}>
                    <div style={{ position:"absolute", left:0, top:3, width:14, height:14, borderRadius:"50%", background: w.status==="done"?COLORS.accent3:w.status==="current"?COLORS.accent:COLORS.border, border:`2px solid ${COLORS.bg}`, boxShadow: w.status==="current"?`0 0 8px ${COLORS.accent}`:"none" }}/>
                    {i < profile.weeklyPlan.length-1 && <div style={{ position:"absolute", left:7, top:18, bottom:-18, width:1, background:COLORS.border }}/>}
                    <div style={{ fontFamily:"monospace", fontSize:10, color:COLORS.accent, marginBottom:3 }}>WEEK {w.week} · {w.focus}</div>
                    <div style={{ fontSize:13, fontWeight:700, marginBottom:6 }}>{w.title}</div>
                    <div style={{ display:"flex", flexDirection:"column", gap:5 }}>
                      {w.tasks.map((t, j) => (
                        <div key={j} style={{ display:"flex", alignItems:"center", gap:8, padding:"7px 10px", borderRadius:7, background:COLORS.bg2, fontSize:12, color:COLORS.text2 }}>
                          <div style={{ width:14, height:14, borderRadius:3, border:`1px solid ${w.status==="done"?COLORS.accent3:COLORS.border}`, background: w.status==="done"?COLORS.accent3:"transparent", display:"flex", alignItems:"center", justifyContent:"center", fontSize:9, color:"white", flexShrink:0 }}>
                            {w.status==="done"?"✓":""}
                          </div>
                          {t}
                        </div>
                      ))}
                    </div>
                  </div>
                ))}
              </div>
            </>
          )}
        </div>

        {/* RIGHT PANEL */}
        <div style={s.rightPanel}>
          <div style={s.rpHeader}>⚠️ Live Alerts {profile && <span style={{ fontFamily:"monospace", fontSize:10, padding:"2px 7px", borderRadius:20, background:COLORS.accent2, color:"white", marginLeft:8 }}>{profile.alerts.length}</span>}</div>
          <div style={s.rpContent}>
            {!profile ? (
              <div style={{ fontSize:11, color:COLORS.text3, fontFamily:"monospace", textAlign:"center", padding:16 }}>Alerts appear after resume analysis</div>
            ) : profile.alerts.map((a, i) => (
              <div key={i} style={s.alert(a.type)}>
                <div style={{ fontSize:14, flexShrink:0 }}>{a.type==="danger"?"🚨":a.type==="warning"?"⚠️":"✅"}</div>
                <div style={{ fontSize:11, lineHeight:1.5, color:COLORS.text2 }}>
                  <strong style={{ color:COLORS.text, display:"block", marginBottom:1, fontSize:12 }}>{a.title}</strong>
                  {a.message}
                </div>
              </div>
            ))}
          </div>
          <div style={{ borderTop:`1px solid ${COLORS.border}`, padding:14 }}>
            <div style={{ fontSize:12, fontWeight:700, marginBottom:10 }}>🏢 Company Matches</div>
            {!profile ? (
              <div style={{ fontSize:11, color:COLORS.text3, fontFamily:"monospace" }}>Upload resume to see matches</div>
            ) : profile.targetCompanies.slice(0,4).map((c, i) => (
              <div key={i} style={s.companyItem}>
                <div style={{ width:32, height:32, borderRadius:7, background:c.color, display:"flex", alignItems:"center", justifyContent:"center", fontSize:12, fontWeight:800, color:"white" }}>{c.logo}</div>
                <div style={{ flex:1 }}>
                  <div style={{ fontSize:12, fontWeight:700 }}>{c.name}</div>
                  <div style={{ fontSize:10, color:COLORS.text2, fontFamily:"monospace" }}>{c.role}</div>
                </div>
                <div style={{ fontFamily:"monospace", fontSize:11, fontWeight:600, color: c.matchScore>=70?COLORS.accent3:c.matchScore>=50?"#ffd700":COLORS.accent2 }}>{c.matchScore}%</div>
              </div>
            ))}
          </div>
        </div>
      </div>

      {/* LOADING OVERLAY */}
      {loading && (
        <div style={{ position:"fixed", inset:0, background:"rgba(10,10,15,0.9)", display:"flex", alignItems:"center", justifyContent:"center", zIndex:500, flexDirection:"column", gap:16 }}>
          <div style={{ width:40, height:40, border:`3px solid ${COLORS.border}`, borderTopColor:COLORS.accent, borderRadius:"50%", animation:"spin 1s linear infinite" }}/>
          <div style={{ fontSize:14, fontWeight:700 }}>{loadingText}</div>
          <div style={{ fontSize:11, color:COLORS.text2, fontFamily:"monospace" }}>Running agentic analysis with Claude AI</div>
        </div>
      )}

      <style>{`
        @keyframes spin { to { transform: rotate(360deg); } }
        @keyframes bounce { 0%,80%,100%{transform:scale(.8);opacity:.5} 40%{transform:scale(1.2);opacity:1} }
        ::-webkit-scrollbar { width: 3px; }
        ::-webkit-scrollbar-thumb { background: #2a2a3a; border-radius: 99px; }
      `}</style>
    </div>
  );
}
