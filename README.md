# weselect

src/main.jsx
import React from 'react';
import ReactDOM from 'react-dom/client';
import { BrowserRouter } from 'react-router-dom';
import App from './App.jsx';
import { UiProvider } from './context/UiContext.jsx';
import { AuthProvider } from './context/AuthContext.jsx';
import './styles.css';

ReactDOM.createRoot(document.getElementById('root')).render(
  <React.StrictMode>
    <BrowserRouter>
      <UiProvider>
        <AuthProvider>
          <App />
        </AuthProvider>
      </UiProvider>
    </BrowserRouter>
  </React.StrictMode>
);

src/api.js
const BASE = '/api/v1';

export class ApiError extends Error {
  constructor(status, code, message, details) {
    super(message);
    this.status = status;
    this.code = code;
    this.details = details;
  }
}

function extractMessage(data, status) {
  if (data?.error?.message) return data.error.message;
  if (typeof data?.detail === 'string') return data.detail;
  if (Array.isArray(data?.detail)) {
    return data.detail.map((d) => d.msg || JSON.stringify(d)).join('; ');
  }
  return `Request failed (${status})`;
}

async function request(path, { method = 'GET', body, formData } = {}) {
  const opts = { method, credentials: 'same-origin', headers: {} };
  if (formData) {
    opts.body = formData;
  } else if (body !== undefined) {
    opts.headers['Content-Type'] = 'application/json';
    opts.body = JSON.stringify(body);
  }

  let res;
  try {
    res = await fetch(BASE + path, opts);
  } catch {
    throw new ApiError(0, 'NETWORK', 'Cannot reach the server. Check your connection and try again.');
  }

  if (res.status === 204) return null;

  let data = null;
  if ((res.headers.get('content-type') || '').includes('application/json')) {
    data = await res.json().catch(() => null);
  }

  if (!res.ok) {
    if (res.status === 401 && !path.startsWith('/auth/login')) {
      window.dispatchEvent(new Event('auth:unauthorized'));
    }
    throw new ApiError(
      res.status,
      data?.error?.code || `HTTP_${res.status}`,
      extractMessage(data, res.status),
      data?.error?.details
    );
  }
  return data;
}

const enc = encodeURIComponent;

export const api = {
  // Auth
  login: (email, password) => request('/auth/login', { method: 'POST', body: { email, password } }),
  logout: () => request('/auth/logout', { method: 'POST' }),
  me: () => request('/auth/me'),

  // Users (admin)
  listUsers: () => request('/users'),
  createUser: (body) => request('/users', { method: 'POST', body }),
  updateUser: (id, body) => request(`/users/${enc(id)}`, { method: 'PATCH', body }),

  // Assessments
  listAssessments: (status) => request('/assessments' + (status ? `?status=${enc(status)}` : '')),
  createAssessment: (body) => request('/assessments', { method: 'POST', body }),
  getAssessment: (id) => request(`/assessments/${enc(id)}`),
  updateAssessment: (id, body) => request(`/assessments/${enc(id)}`, { method: 'PATCH', body }),
  deleteAssessment: (id) => request(`/assessments/${enc(id)}`, { method: 'DELETE' }),
  closeAssessment: (id) => request(`/assessments/${enc(id)}/close`, { method: 'POST' }),
  uploadJd: (id, file) => {
    const fd = new FormData();
    fd.append('file', file);
    return request(`/assessments/${enc(id)}/jd-file`, { method: 'POST', formData: fd });
  },

  // AI jobs + questions
  generate: (id, body = {}) => request(`/assessments/${enc(id)}/generate`, { method: 'POST', body }),
  getJob: (jobId) => request(`/jobs/${enc(jobId)}`),
  listQuestions: (id) => request(`/assessments/${enc(id)}/questions`),
  addQuestion: (id, body) => request(`/assessments/${enc(id)}/questions`, { method: 'POST', body }),
  updateQuestion: (qid, body) => request(`/questions/${enc(qid)}`, { method: 'PATCH', body }),
  deleteQuestion: (qid) => request(`/questions/${enc(qid)}`, { method: 'DELETE' }),
  regenerateQuestion: (qid) => request(`/questions/${enc(qid)}/regenerate`, { method: 'POST' }),
  approve: (id) => request(`/assessments/${enc(id)}/approve`, { method: 'POST' }),

  // Participants
  listParticipants: (id) => request(`/assessments/${enc(id)}/participants`),
  addParticipants: (id, body) => request(`/assessments/${enc(id)}/participants`, { method: 'POST', body }),
  deleteParticipant: (pid) => request(`/participants/${enc(pid)}`, { method: 'DELETE' }),
  sendInvitations: (id, body = {}) => request(`/assessments/${enc(id)}/invitations`, { method: 'POST', body }),
  resend: (pid) => request(`/participants/${enc(pid)}/resend`, { method: 'POST' }),

  // Results
  results: (id, sort = 'score') => request(`/assessments/${enc(id)}/results?sort=${enc(sort)}`),
  getAttempt: (attemptId) => request(`/attempts/${enc(attemptId)}`),
  override: (attemptId, qid, body) =>
    request(`/attempts/${enc(attemptId)}/scores/${enc(qid)}/override`, { method: 'POST', body }),
};

// Poll a background job (AI work) every 2 s until done / failed / timeout.
export async function waitForJob(jobId, { timeoutMs = 180000 } = {}) {
  const start = Date.now();
  // eslint-disable-next-line no-constant-condition
  while (true) {
    const job = await api.getJob(jobId);
    if (job.status === 'done') return job;
    if (job.status === 'failed') {
      throw new ApiError(422, 'JOB_FAILED', job.error || 'The AI job failed. Please try again.');
    }
    if (Date.now() - start > timeoutMs) {
      throw new ApiError(504, 'JOB_TIMEOUT', 'The AI job is taking too long. Please check again in a minute.');
    }
    await new Promise((r) => setTimeout(r, 2000));
  }
}

// Download CSV / PDF export as a file (works with the cookie session).
export async function downloadExport(id, format) {
  let res;
  try {
    res = await fetch(`${BASE}/assessments/${enc(id)}/export?format=${format}`, { credentials: 'same-origin' });
  } catch {
    throw new ApiError(0, 'NETWORK', 'Cannot reach the server.');
  }
  if (!res.ok) {
    let data = null;
    try { data = await res.json(); } catch { /* not json */ }
    throw new ApiError(res.status, data?.error?.code || `HTTP_${res.status}`, extractMessage(data, res.status));
  }
  const blob = await res.blob();
  const cd = res.headers.get('content-disposition') || '';
  const match = /filename\*?=(?:UTF-8'')?"?([^";]+)"?/i.exec(cd);
  const filename = match ? decodeURIComponent(match[1]) : `results.${format}`;
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url;
  a.download = filename;
  document.body.appendChild(a);
  a.click();
  a.remove();
  URL.revokeObjectURL(url);
}

src/utils.js
export function formatDate(iso) {
  if (!iso) return '-';
  const d = new Date(iso);
  if (Number.isNaN(d.getTime())) return '-';
  return d.toLocaleString(undefined, { dateStyle: 'medium', timeStyle: 'short' });
}

export function formatDuration(sec) {
  if (sec === null || sec === undefined) return '-';
  const m = Math.floor(sec / 60);
  const s = Math.round(sec % 60);
  return `${m}m ${String(s).padStart(2, '0')}s`;
}

export function wordCount(text) {
  const t = (text || '').trim();
  return t ? t.split(/\s+/).length : 0;
}

export function fmtNum(n, digits = 1) {
  if (n === null || n === undefined || Number.isNaN(Number(n))) return '-';
  return Number(n).toFixed(digits).replace(/\.0+$/, '');
}

export const TYPE_LABEL = { mcq: 'MCQ', descriptive: 'Descriptive', cognitive: 'Cognitive' };


src/context/AuthContext.jsx
import { createContext, useCallback, useContext, useEffect, useState } from 'react';
import { api } from '../api.js';

const AuthCtx = createContext(null);

export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [tenant, setTenant] = useState(null);
  const [loading, setLoading] = useState(true);

  useEffect(() => {
    let active = true;
    api.me()
      .then((d) => { if (active) { setUser(d.user); setTenant(d.tenant); } })
      .catch(() => {})
      .finally(() => { if (active) setLoading(false); });
    return () => { active = false; };
  }, []);

  // Any 401 from the API (expired session) sends the user back to login.
  useEffect(() => {
    const handler = () => { setUser(null); setTenant(null); };
    window.addEventListener('auth:unauthorized', handler);
    return () => window.removeEventListener('auth:unauthorized', handler);
  }, []);

  const login = useCallback(async (email, password) => {
    const d = await api.login(email, password);
    setUser(d.user);
    setTenant(d.tenant);
  }, []);

  const logout = useCallback(async () => {
    try { await api.logout(); } catch { /* ignore */ }
    setUser(null);
    setTenant(null);
  }, []);

  return (
    <AuthCtx.Provider value={{ user, tenant, loading, login, logout }}>
      {children}
    </AuthCtx.Provider>
  );
}

export const useAuth = () => useContext(AuthCtx);

src/context/UiContext.jsx
import { createContext, useCallback, useContext, useRef, useState } from 'react';
import Modal from '../components/Modal.jsx';

const UiCtx = createContext(null);

export function UiProvider({ children }) {
  const [toasts, setToasts] = useState([]);
  const [dialog, setDialog] = useState(null);
  const idRef = useRef(0);

  const toast = useCallback((message, type = 'info') => {
    const id = ++idRef.current;
    setToasts((t) => [...t, { id, message, type }]);
    setTimeout(() => setToasts((t) => t.filter((x) => x.id !== id)), 5000);
  }, []);

  // confirm({title, message, confirmText, danger}) -> Promise<boolean>
  const confirm = useCallback(
    (opts) => new Promise((resolve) => setDialog({ ...opts, resolve })),
    []
  );

  const close = (value) => {
    dialog.resolve(value);
    setDialog(null);
  };

  return (
    <UiCtx.Provider value={{ toast, confirm }}>
      {children}
      <div className="toasts" role="status" aria-live="polite">
        {toasts.map((t) => (
          <div key={t.id} className={`toast toast-${t.type}`}>{t.message}</div>
        ))}
      </div>
      {dialog && (
        <Modal
          title={dialog.title || 'Please confirm'}
          onClose={() => close(false)}
          size="sm"
          footer={
            <>
              <button className="btn btn-ghost" onClick={() => close(false)}>Cancel</button>
              <button
                className={`btn ${dialog.danger ? 'btn-danger' : 'btn-primary'}`}
                onClick={() => close(true)}
              >
                {dialog.confirmText || 'Confirm'}
              </button>
            </>
          }
        >
          <p>{dialog.message}</p>
        </Modal>
      )}
    </UiCtx.Provider>
  );
}

export const useUi = () => useContext(UiCtx);

src/components/Modal.jsx
import { useEffect } from 'react';

export default function Modal({ title, children, footer, onClose, size = 'md' }) {
  useEffect(() => {
    const onKey = (e) => { if (e.key === 'Escape') onClose(); };
    document.addEventListener('keydown', onKey);
    return () => document.removeEventListener('keydown', onKey);
  }, [onClose]);

  return (
    <div className="modal-overlay" onMouseDown={(e) => { if (e.target === e.currentTarget) onClose(); }}>
      <div className={`modal modal-${size}`} role="dialog" aria-modal="true" aria-label={title}>
        <div className="modal-head">
          <h3>{title}</h3>
          <button className="icon-btn" onClick={onClose} aria-label="Close">&times;</button>
        </div>
        <div className="modal-body">{children}</div>
        {footer && <div className="modal-foot">{footer}</div>}
      </div>
    </div>
  );
}

src/components/Badge.jsx
export default function Badge({ value }) {
  const v = value || 'none';
  return <span className={`badge b-${v}`}>{String(v).replace(/_/g, ' ')}</span>;
}

src/components/PageState.jsx
export function Loading({ full = false, text = 'Loading...' }) {
  return (
    <div className={full ? 'state state-full' : 'state'}>
      <div className="spinner" />
      <p>{text}</p>
    </div>
  );
}

export function ErrorBox({ error, onRetry }) {
  return (
    <div className="state">
      <div className="state-icon">!</div>
      <p>{error?.message || 'Something went wrong. Please try again.'}</p>
      {onRetry && <button className="btn btn-primary" onClick={onRetry}>Retry</button>}
    </div>
  );
}

export function Empty({ title, text, action }) {
  return (
    <div className="state">
      <div className="state-icon state-icon-soft">&#9776;</div>
      <h3>{title}</h3>
      {text && <p>{text}</p>}
      {action}
    </div>
  );
}

src/components/Layout.jsx
import { NavLink, Outlet, useNavigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext.jsx';

export default function Layout() {
  const { user, tenant, logout } = useAuth();
  const navigate = useNavigate();

  const onLogout = async () => {
    await logout();
    navigate('/login', { replace: true });
  };

  return (
    <div className="shell">
      <aside className="sidebar">
        <div className="brand">
          <span className="brand-mark">W</span>
          <div>
            <div className="brand-name">WeSelect</div>
            <div className="brand-sub">Assessment Platform</div>
          </div>
        </div>
        <nav className="nav">
          <NavLink to="/assessments" end={false} className={({ isActive }) => (isActive ? 'active' : '')}>
            Assessments
          </NavLink>
          {user?.role === 'admin' && (
            <NavLink to="/users" className={({ isActive }) => (isActive ? 'active' : '')}>
              Team &amp; Users
            </NavLink>
          )}
        </nav>
        <div className="sidebar-foot">{tenant?.name}</div>
      </aside>

      <div className="main">
        <header className="topbar">
          <div />
          <div className="topbar-user">
            <div className="who">
              <strong>{user?.name}</strong>
              <span className="role-chip">{user?.role}</span>
            </div>
            <button className="btn btn-ghost btn-sm" onClick={onLogout}>Sign out</button>
          </div>
        </header>
        <main className="content">
          <Outlet />
        </main>
      </div>
    </div>
  );
}

src/components/AssessmentForm.jsx
import { useState } from 'react';
import { wordCount } from '../utils.js';

const newSection = () => ({ type: 'mcq', domain: '', difficulty: 'medium', count: 5 });

export default function AssessmentForm({ initial, onSubmit, submitLabel, saving }) {
  const [title, setTitle] = useState(initial?.title || '');
  const [durationMinutes, setDuration] = useState(initial?.durationMinutes ?? 60);
  const [passingScore, setPassing] = useState(initial?.passingScore ?? 60);
  const [jobDescription, setJd] = useState(initial?.jobDescription || '');
  const [sections, setSections] = useState(
    initial?.sections?.length ? initial.sections.map((s) => ({ ...s })) : [newSection()]
  );
  const [file, setFile] = useState(null);
  const [errors, setErrors] = useState({});

  const words = wordCount(jobDescription);
  const total = sections.reduce((a, s) => a + (Number(s.count) || 0), 0);

  const updateSection = (i, patch) =>
    setSections((arr) => arr.map((s, idx) => (idx === i ? { ...s, ...patch } : s)));

  const onFile = (e) => {
    const f = e.target.files?.[0];
    if (!f) return setFile(null);
    const ok = /\.(pdf|txt)$/i.test(f.name);
    if (!ok) { setErrors((er) => ({ ...er, file: 'Only .pdf or .txt files are allowed.' })); e.target.value = ''; return; }
    if (f.size > 2 * 1024 * 1024) { setErrors((er) => ({ ...er, file: 'File must be 2 MB or smaller.' })); e.target.value = ''; return; }
    setErrors((er) => ({ ...er, file: undefined }));
    setFile(f);
  };

  const validate = () => {
    const er = {};
    if (!title.trim()) er.title = 'Title is required.';
    const d = Number(durationMinutes);
    if (!Number.isFinite(d) || d < 1 || d > 480) er.duration = 'Enter 1 to 480 minutes.';
    const p = Number(passingScore);
    if (!Number.isFinite(p) || p < 0 || p > 100) er.passing = 'Enter a percentage from 0 to 100.';
    if (jobDescription.length > 20000) er.jd = 'Job description is limited to 20,000 characters.';
    if (!sections.length) er.sections = 'Add at least one section.';
    sections.forEach((s, i) => {
      if (!s.domain.trim()) er[`domain${i}`] = 'Required';
      const c = Number(s.count);
      if (!Number.isInteger(c) || c < 1 || c > 20) er[`count${i}`] = '1 to 20';
    });
    if (total > 40) er.sections = 'Total questions cannot exceed 40.';
    setErrors(er);
    return Object.keys(er).length === 0;
  };

  const submit = (e) => {
    e.preventDefault();
    if (!validate()) return;
    onSubmit(
      {
        title: title.trim(),
        jobDescription,
        durationMinutes: Number(durationMinutes),
        passingScore: Number(passingScore),
        sections: sections.map((s) => ({
          type: s.type,
          domain: s.domain.trim(),
          difficulty: s.difficulty,
          count: Number(s.count),
        })),
      },
      file
    );
  };

  return (
    <form onSubmit={submit} noValidate>
      <div className="card">
        <h3 className="card-title">Basic information</h3>
        <div className="form-group">
          <label>Assessment title</label>
          <input className="input" value={title} onChange={(e) => setTitle(e.target.value)} placeholder="e.g. Backend Engineer Screening" maxLength={200} />
          {errors.title && <div className="field-error">{errors.title}</div>}
        </div>
        <div className="form-row">
          <div className="form-group">
            <label>Duration (minutes)</label>
            <input className="input" type="number" value={durationMinutes} onChange={(e) => setDuration(e.target.value)} />
            {errors.duration && <div className="field-error">{errors.duration}</div>}
          </div>
          <div className="form-group">
            <label>Passing score (%)</label>
            <input className="input" type="number" value={passingScore} onChange={(e) => setPassing(e.target.value)} />
            {errors.passing && <div className="field-error">{errors.passing}</div>}
          </div>
        </div>
      </div>

      <div className="card">
        <h3 className="card-title">Job description</h3>
        <p className="muted">Paste the job description, or upload a PDF/TXT file (max 2 MB). At least 100 words are needed to generate questions.</p>
        <div className="form-group">
          <textarea className="input" rows={9} value={jobDescription} onChange={(e) => setJd(e.target.value)} placeholder="Paste the full job description here..." />
          <div className={`hint ${words > 0 && words < 100 ? 'hint-warn' : ''}`}>
            {words} words &middot; {jobDescription.length}/20000 characters
            {words > 0 && words < 100 ? ' (needs at least 100 words)' : ''}
          </div>
          {errors.jd && <div className="field-error">{errors.jd}</div>}
        </div>
        <div className="form-group">
          <label>Or upload a file</label>
          <input type="file" accept=".pdf,.txt" onChange={onFile} />
          {initial?.jdFileName && !file && <div className="hint">Current file: {initial.jdFileName}</div>}
          {file && <div className="hint">Selected: {file.name}. Its extracted text will replace the pasted text.</div>}
          {errors.file && <div className="field-error">{errors.file}</div>}
        </div>
      </div>

      <div className="card">
        <div className="card-head">
          <h3 className="card-title">Question sections</h3>
          <span className="muted">Total: {total} / 40</span>
        </div>
        {sections.map((s, i) => (
          <div className="section-row" key={i}>
            <div className="form-group">
              <label>Type</label>
              <select className="input" value={s.type} onChange={(e) => updateSection(i, { type: e.target.value })}>
                <option value="mcq">MCQ</option>
                <option value="descriptive">Descriptive</option>
                <option value="cognitive">Cognitive</option>
              </select>
            </div>
            <div className="form-group grow">
              <label>Domain / topic</label>
              <input className="input" value={s.domain} onChange={(e) => updateSection(i, { domain: e.target.value })} placeholder="e.g. Node.js" />
              {errors[`domain${i}`] && <div className="field-error">{errors[`domain${i}`]}</div>}
            </div>
            <div className="form-group">
              <label>Difficulty</label>
              <select className="input" value={s.difficulty} onChange={(e) => updateSection(i, { difficulty: e.target.value })}>
                <option value="easy">Easy</option>
                <option value="medium">Medium</option>
                <option value="hard">Hard</option>
              </select>
            </div>
            <div className="form-group narrow">
              <label>Count</label>
              <input className="input" type="number" value={s.count} onChange={(e) => updateSection(i, { count: e.target.value })} />
              {errors[`count${i}`] && <div className="field-error">{errors[`count${i}`]}</div>}
            </div>
            <button type="button" className="btn btn-ghost btn-sm remove-btn" disabled={sections.length === 1} onClick={() => setSections((a) => a.filter((_, idx) => idx !== i))}>
              Remove
            </button>
          </div>
        ))}
        {errors.sections && <div className="field-error">{errors.sections}</div>}
        <button type="button" className="btn btn-outline" onClick={() => setSections((a) => [...a, newSection()])}>
          + Add section
        </button>
      </div>

      <div className="form-actions">
        <button className="btn btn-primary" disabled={saving}>{saving ? 'Saving...' : submitLabel}</button>
      </div>
    </form>
  );
}

src/components/QuestionEditor.jsx: used to edit and to add custom questions:
import { useState } from 'react';

const LETTERS = ['A', 'B', 'C', 'D'];

export default function QuestionEditor({ initial, onSave, onCancel, saving }) {
  const editing = Boolean(initial);
  const [type, setType] = useState(initial?.type || 'mcq');
  const [body, setBody] = useState(initial?.body || '');
  const [options, setOptions] = useState(
    initial?.options?.length === 4 ? [...initial.options] : ['', '', '', '']
  );
  const [correct, setCorrect] = useState(initial?.correct || 'A');
  const [rubric, setRubric] = useState(initial?.rubric || '');
  const [maxScore, setMaxScore] = useState(initial?.maxScore ?? (initial?.type === 'descriptive' ? 10 : 1));
  const [error, setError] = useState('');

  const isChoice = type === 'mcq' || type === 'cognitive';

  const submit = () => {
    if (!body.trim()) return setError('Question text is required.');
    const ms = Number(maxScore);
    if (!Number.isFinite(ms) || ms <= 0) return setError('Max score must be greater than 0.');
    if (isChoice) {
      if (options.some((o) => !o.trim())) return setError('All four options are required.');
      onSave({ type, body: body.trim(), options: options.map((o) => o.trim()), correct, maxScore: ms });
    } else {
      if (!rubric.trim()) return setError('A scoring rubric is required for descriptive questions.');
      onSave({ type, body: body.trim(), rubric: rubric.trim(), maxScore: ms });
    }
  };

  return (
    <div>
      {!editing && (
        <div className="form-group">
          <label>Question type</label>
          <select
            className="input"
            value={type}
            onChange={(e) => { setType(e.target.value); setMaxScore(e.target.value === 'descriptive' ? 10 : 1); }}
          >
            <option value="mcq">MCQ</option>
            <option value="descriptive">Descriptive</option>
            <option value="cognitive">Cognitive</option>
          </select>
        </div>
      )}
      <div className="form-group">
        <label>Question</label>
        <textarea className="input" rows={3} value={body} onChange={(e) => setBody(e.target.value)} />
      </div>

      {isChoice ? (
        <>
          {LETTERS.map((L, i) => (
            <div className="form-group" key={L}>
              <label>Option {L}</label>
              <input className="input" value={options[i]} onChange={(e) => setOptions((o) => o.map((v, idx) => (idx === i ? e.target.value : v)))} />
            </div>
          ))}
          <div className="form-row">
            <div className="form-group">
              <label>Correct answer</label>
              <select className="input" value={correct} onChange={(e) => setCorrect(e.target.value)}>
                {LETTERS.map((L) => <option key={L} value={L}>{L}</option>)}
              </select>
            </div>
            <div className="form-group">
              <label>Marks</label>
              <input className="input" type="number" value={maxScore} onChange={(e) => setMaxScore(e.target.value)} />
            </div>
          </div>
        </>
      ) : (
        <>
          <div className="form-group">
            <label>Scoring rubric</label>
            <textarea className="input" rows={4} value={rubric} onChange={(e) => setRubric(e.target.value)} placeholder="e.g. 4 marks for explaining X, 3 marks for Y, 3 marks for examples" />
          </div>
          <div className="form-group">
            <label>Max score</label>
            <input className="input" type="number" value={maxScore} onChange={(e) => setMaxScore(e.target.value)} />
          </div>
        </>
      )}

      {error && <div className="field-error">{error}</div>}
      <div className="modal-actions">
        <button className="btn btn-ghost" onClick={onCancel}>Cancel</button>
        <button className="btn btn-primary" onClick={submit} disabled={saving}>{saving ? 'Saving...' : 'Save question'}</button>
      </div>
    </div>
  );
}

src/pages/LoginPage.jsx
import { useState } from 'react';
import { useLocation, useNavigate } from 'react-router-dom';
import { useAuth } from '../context/AuthContext.jsx';

export default function LoginPage() {
  const { login } = useAuth();
  const navigate = useNavigate();
  const location = useLocation();
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [error, setError] = useState('');
  const [busy, setBusy] = useState(false);

  const submit = async (e) => {
    e.preventDefault();
    setError('');
    if (!email.trim() || !password) return setError('Enter your email and password.');
    setBusy(true);
    try {
      await login(email.trim(), password);
      navigate(location.state?.from || '/assessments', { replace: true });
    } catch (err) {
      if (err.status === 429) setError('Too many failed attempts. Please wait 10 minutes and try again.');
      else if (err.status === 401) setError('Invalid email or password.');
      else setError(err.message);
    } finally {
      setBusy(false);
    }
  };

  return (
    <div className="login-wrap">
      <div className="login-hero">
        <div className="brand-mark brand-mark-lg">W</div>
        <h1>WeSelect</h1>
        <p>AI-powered candidate assessments. Create role-specific tests, invite candidates, and review ranked results, all in one place.</p>
      </div>
      <div className="login-panel">
        <form className="login-card" onSubmit={submit} noValidate>
          <h2>Sign in</h2>
          <p className="muted">Recruiter and admin access</p>
          <div className="form-group">
            <label>Email</label>
            <input className="input" type="email" autoComplete="username" value={email} onChange={(e) => setEmail(e.target.value)} autoFocus />
          </div>
          <div className="form-group">
            <label>Password</label>
            <input className="input" type="password" autoComplete="current-password" value={password} onChange={(e) => setPassword(e.target.value)} />
          </div>
          {error && <div className="alert alert-error">{error}</div>}
          <button className="btn btn-primary btn-block" disabled={busy}>{busy ? 'Signing in...' : 'Sign in'}</button>
        </form>
      </div>
    </div>
  );
}

src/pages/AssessmentsPage.jsx: the dashboard:
import { useCallback, useEffect, useMemo, useState } from 'react';
import { Link, useNavigate } from 'react-router-dom';
import { api } from '../api.js';
import Badge from '../components/Badge.jsx';
import { Empty, ErrorBox, Loading } from '../components/PageState.jsx';
import { useUi } from '../context/UiContext.jsx';
import { formatDate } from '../utils.js';

export default function AssessmentsPage() {
  const [items, setItems] = useState(null);
  const [error, setError] = useState(null);
  const [status, setStatus] = useState('');
  const navigate = useNavigate();
  const { toast, confirm } = useUi();

  const load = useCallback(async () => {
    try {
      setError(null);
      setItems(await api.listAssessments(status));
    } catch (e) { setError(e); }
  }, [status]);

  useEffect(() => { setItems(null); load(); }, [load]);

  const stats = useMemo(() => {
    const list = items || [];
    return {
      total: list.length,
      open: list.filter((a) => a.status === 'open').length,
      participants: list.reduce((n, a) => n + (a.participantCount || 0), 0),
      submitted: list.reduce((n, a) => n + (a.submittedCount || 0), 0),
    };
  }, [items]);

  const remove = async (a) => {
    const ok = await confirm({ title: 'Delete assessment', message: `Delete draft "${a.title}"? This cannot be undone.`, confirmText: 'Delete', danger: true });
    if (!ok) return;
    try { await api.deleteAssessment(a.id); toast('Assessment deleted.', 'success'); load(); }
    catch (e) { toast(e.message, 'error'); }
  };

  return (
    <>
      <div className="page-header">
        <div>
          <h1>Assessments</h1>
          <p className="muted">Create, manage and review your candidate assessments.</p>
        </div>
        <Link className="btn btn-primary" to="/assessments/new">+ New assessment</Link>
      </div>

      <div className="stats">
        <div className="stat"><span>Assessments</span><strong>{stats.total}</strong></div>
        <div className="stat"><span>Open</span><strong>{stats.open}</strong></div>
        <div className="stat"><span>Participants</span><strong>{stats.participants}</strong></div>
        <div className="stat"><span>Submitted</span><strong>{stats.submitted}</strong></div>
      </div>

      <div className="card">
        <div className="card-head">
          <h3 className="card-title">All assessments</h3>
          <select className="input input-auto" value={status} onChange={(e) => setStatus(e.target.value)}>
            <option value="">All statuses</option>
            <option value="draft">Draft</option>
            <option value="approved">Approved</option>
            <option value="open">Open</option>
            <option value="closed">Closed</option>
          </select>
        </div>

        {error ? <ErrorBox error={error} onRetry={load} />
          : !items ? <Loading />
          : items.length === 0 ? (
            <Empty title="No assessments yet" text="Create your first assessment to get started."
              action={<Link className="btn btn-primary" to="/assessments/new">Create assessment</Link>} />
          ) : (
            <div className="table-wrap">
              <table className="table">
                <thead>
                  <tr><th>Title</th><th>Status</th><th>Participants</th><th>Submitted</th><th>Created</th><th /></tr>
                </thead>
                <tbody>
                  {items.map((a) => (
                    <tr key={a.id} className="clickable" onClick={() => navigate(`/assessments/${a.id}`)}>
                      <td><strong>{a.title}</strong></td>
                      <td><Badge value={a.status} /></td>
                      <td>{a.participantCount ?? 0}</td>
                      <td>{a.submittedCount ?? 0}</td>
                      <td>{formatDate(a.createdAt)}</td>
                      <td className="right" onClick={(e) => e.stopPropagation()}>
                        {a.status === 'draft' && (
                          <button className="btn btn-ghost btn-sm" onClick={() => remove(a)}>Delete</button>
                        )}
                      </td>
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
          )}
      </div>
    </>
  );
}

src/pages/NewAssessmentPage.jsx
import { useState } from 'react';
import { useNavigate } from 'react-router-dom';
import { api } from '../api.js';
import AssessmentForm from '../components/AssessmentForm.jsx';
import { useUi } from '../context/UiContext.jsx';

export default function NewAssessmentPage() {
  const navigate = useNavigate();
  const { toast } = useUi();
  const [saving, setSaving] = useState(false);

  const onSubmit = async (values, file) => {
    setSaving(true);
    let created;
    try {
      created = await api.createAssessment(values);
    } catch (e) {
      toast(e.message, 'error');
      setSaving(false);
      return;
    }
    if (file) {
      try {
        await api.uploadJd(created.id, file);
      } catch (e) {
        toast(`Assessment created, but the file upload failed: ${e.message}`, 'error');
      }
    }
    toast('Assessment created. Now generate questions.', 'success');
    navigate(`/assessments/${created.id}/questions`);
  };

  return (
    <>
      <div className="page-header">
        <div>
          <h1>New assessment</h1>
          <p className="muted">Describe the role. AI will draft questions that you can review and edit.</p>
        </div>
      </div>
      <AssessmentForm onSubmit={onSubmit} submitLabel="Create assessment" saving={saving} />
    </>
  );
}

src/pages/AssessmentLayout.jsx
import { useCallback, useEffect, useState } from 'react';
import { NavLink, Outlet, useNavigate, useParams } from 'react-router-dom';
import { api } from '../api.js';
import Badge from '../components/Badge.jsx';
import { ErrorBox, Loading } from '../components/PageState.jsx';
import { useUi } from '../context/UiContext.jsx';

export default function AssessmentLayout() {
  const { id } = useParams();
  const navigate = useNavigate();
  const { toast, confirm } = useUi();
  const [assessment, setAssessment] = useState(null);
  const [error, setError] = useState(null);

  const load = useCallback(async () => {
    try {
      const a = await api.getAssessment(id);
      setAssessment(a);
      setError(null);
    } catch (e) { setError(e); }
  }, [id]);

  useEffect(() => { setAssessment(null); load(); }, [load]);

  if (error) return <ErrorBox error={error} onRetry={load} />;
  if (!assessment) return <Loading />;

  const closeAssessment = async () => {
    const ok = await confirm({
      title: 'Close assessment',
      message: 'No new attempts will be allowed and invitations will expire. Continue?',
      confirmText: 'Close assessment', danger: true,
    });
    if (!ok) return;
    try { await api.closeAssessment(id); toast('Assessment closed.', 'success'); load(); }
    catch (e) { toast(e.message, 'error'); }
  };

  const tab = ({ isActive }) => (isActive ? 'tab active' : 'tab');

  return (
    <>
      <button className="link-back" onClick={() => navigate('/assessments')}>&larr; All assessments</button>
      <div className="page-header">
        <div>
          <h1>{assessment.title}</h1>
          <div className="meta-line">
            <Badge value={assessment.status} />
            <span className="muted">{assessment.durationMinutes} min &middot; pass {assessment.passingScore}%</span>
          </div>
        </div>
        {(assessment.status === 'approved' || assessment.status === 'open') && (
          <button className="btn btn-outline" onClick={closeAssessment}>Close assessment</button>
        )}
      </div>

      <div className="tabs">
        <NavLink end to={`/assessments/${id}`} className={tab}>Overview</NavLink>
        <NavLink to={`/assessments/${id}/questions`} className={tab}>Questions</NavLink>
        <NavLink to={`/assessments/${id}/participants`} className={tab}>Participants</NavLink>
        <NavLink to={`/assessments/${id}/results`} className={tab}>Results</NavLink>
      </div>

      <Outlet context={{ assessment, reload: load }} />
    </>
  );
}

src/pages/OverviewTab.jsx
import { useState } from 'react';
import { useOutletContext } from 'react-router-dom';
import { api } from '../api.js';
import AssessmentForm from '../components/AssessmentForm.jsx';
import { TYPE_LABEL, wordCount } from '../utils.js';
import { useUi } from '../context/UiContext.jsx';

export default function OverviewTab() {
  const { assessment, reload } = useOutletContext();
  const { toast } = useUi();
  const [saving, setSaving] = useState(false);
  const [formKey, setFormKey] = useState(0);

  if (assessment.status !== 'draft') {
    return (
      <div className="card">
        <h3 className="card-title">Assessment details</h3>
        <p className="muted">This assessment is {assessment.status}, so its settings can no longer be edited.</p>
        <dl className="details">
          <dt>Duration</dt><dd>{assessment.durationMinutes} minutes</dd>
          <dt>Passing score</dt><dd>{assessment.passingScore}%</dd>
          <dt>Sections</dt>
          <dd>
            {(assessment.sections || []).map((s, i) => (
              <div key={i}>{TYPE_LABEL[s.type] || s.type} &middot; {s.domain} &middot; {s.difficulty} &middot; {s.count} questions</div>
            ))}
          </dd>
          <dt>Job description</dt>
          <dd>{wordCount(assessment.jobDescription)} words{assessment.jdFileName ? ` (file: ${assessment.jdFileName})` : ''}</dd>
        </dl>
        <pre className="jd-box">{assessment.jobDescription}</pre>
      </div>
    );
  }

  const onSubmit = async (values, file) => {
    setSaving(true);
    try {
      await api.updateAssessment(assessment.id, values);
      if (file) await api.uploadJd(assessment.id, file);
      toast('Assessment saved.', 'success');
      await reload();
      setFormKey((k) => k + 1);
    } catch (e) {
      toast(e.message, 'error');
    } finally {
      setSaving(false);
    }
  };

  return <AssessmentForm key={formKey} initial={assessment} onSubmit={onSubmit} submitLabel="Save changes" saving={saving} />;
}

**src/pages/QuestionsTab.jsx**
import { useCallback, useEffect, useState } from 'react';
import { useOutletContext } from 'react-router-dom';
import { api, waitForJob } from '../api.js';
import Modal from '../components/Modal.jsx';
import QuestionEditor from '../components/QuestionEditor.jsx';
import { Empty, ErrorBox, Loading } from '../components/PageState.jsx';
import { useUi } from '../context/UiContext.jsx';
import { TYPE_LABEL, wordCount } from '../utils.js';

export default function QuestionsTab() {
  const { assessment, reload } = useOutletContext();
  const { toast, confirm } = useUi();
  const draft = assessment.status === 'draft';

  const [questions, setQuestions] = useState(null);
  const [error, setError] = useState(null);
  const [busy, setBusy] = useState('');
  const [editing, setEditing] = useState(null);   // question being edited
  const [adding, setAdding] = useState(false);
  const [saving, setSaving] = useState(false);

  const load = useCallback(async () => {
    try {
      const q = await api.listQuestions(assessment.id);
      setQuestions([...q].sort((a, b) => (a.position ?? 0) - (b.position ?? 0)));
      setError(null);
    } catch (e) { setError(e); }
  }, [assessment.id]);

  useEffect(() => { load(); }, [load]);

  const runJob = async (start, label, okMsg) => {
    setBusy(label);
    try {
      const { jobId } = await start();
      await waitForJob(jobId);
      toast(okMsg, 'success');
      await load();
    } catch (e) {
      toast(e.message, 'error');
    } finally {
      setBusy('');
    }
  };

  const generate = () => {
    const words = wordCount(assessment.jobDescription);
    if (words < 100) {
      return toast('The job description needs at least 100 words. Edit it on the Overview tab first.', 'error');
    }
    runJob(() => api.generate(assessment.id, {}), 'Generating questions with AI. This can take up to a minute...', 'Questions generated. Please review them.');
  };

  const regenerate = async (q) => {
    const ok = await confirm({ title: 'Regenerate question', message: 'Replace this question with a new AI-generated one?', confirmText: 'Regenerate' });
    if (!ok) return;
    runJob(() => api.regenerateQuestion(q.id), 'Regenerating question...', 'Question regenerated.');
  };

  const remove = async (q) => {
    const ok = await confirm({ title: 'Remove question', message: 'Remove this question permanently?', confirmText: 'Remove', danger: true });
    if (!ok) return;
    try { await api.deleteQuestion(q.id); toast('Question removed.', 'success'); load(); }
    catch (e) { toast(e.message, 'error'); }
  };

  const saveEdit = async (payload) => {
    setSaving(true);
    try { await api.updateQuestion(editing.id, payload); toast('Question updated.', 'success'); setEditing(null); load(); }
    catch (e) { toast(e.message, 'error'); }
    finally { setSaving(false); }
  };

  const saveNew = async (payload) => {
    setSaving(true);
    try { await api.addQuestion(assessment.id, payload); toast('Custom question added.', 'success'); setAdding(false); load(); }
    catch (e) { toast(e.message, 'error'); }
    finally { setSaving(false); }
  };

  const approve = async () => {
    const ok = await confirm({
      title: 'Approve questions',
      message: 'After approval the questions become read-only and you can invite participants. Continue?',
      confirmText: 'Approve',
    });
    if (!ok) return;
    try {
      await api.approve(assessment.id);
      toast('Assessment approved. You can now invite participants.', 'success');
      await reload();
      await load();
    } catch (e) {
      const map = { NO_QUESTIONS: 'Add at least one question before approving.', SECTION_INCOMPLETE: 'Every section needs its questions before approval.' };
      toast(map[e.code] || e.message, 'error');
    }
  };

  if (error) return <ErrorBox error={error} onRetry={load} />;
  if (!questions) return <Loading />;

  return (
    <>
      {busy && (
        <div className="alert alert-info"><div className="spinner spinner-sm" /> {busy}</div>
      )}

      {draft && (
        <div className="toolbar">
          <button className="btn btn-primary" onClick={generate} disabled={Boolean(busy)}>
            {questions.length ? 'Generate more with AI' : 'Generate questions with AI'}
          </button>
          <button className="btn btn-outline" onClick={() => setAdding(true)} disabled={Boolean(busy)}>+ Add custom question</button>
          <span className="grow" />
          <button className="btn btn-success" onClick={approve} disabled={Boolean(busy) || questions.length === 0}>Approve all &amp; continue</button>
        </div>
      )}

      {questions.length === 0 ? (
        <div className="card">
          <Empty title="No questions yet" text={draft ? 'Generate questions with AI or add your own.' : 'This assessment has no questions.'} />
        </div>
      ) : (
        questions.map((q, i) => (
          <div className="card qcard" key={q.id}>
            <div className="qhead">
              <div className="qmeta">
                <span className="qnum">Q{i + 1}</span>
                <span className="badge b-approved">{TYPE_LABEL[q.type] || q.type}</span>
                <span className={`badge ${q.source === 'custom' ? 'b-open' : 'b-none'}`}>{q.source === 'custom' ? 'Custom' : 'AI'}</span>
                <span className="muted">{q.maxScore} {q.maxScore === 1 ? 'mark' : 'marks'}</span>
              </div>
              {draft && (
                <div className="qactions">
                  <button className="btn btn-ghost btn-sm" disabled={Boolean(busy)} onClick={() => setEditing(q)}>Edit</button>
                  {q.source !== 'custom' && (
                    <button className="btn btn-ghost btn-sm" disabled={Boolean(busy)} onClick={() => regenerate(q)}>Regenerate</button>
                  )}
                  <button className="btn btn-ghost btn-sm danger-text" disabled={Boolean(busy)} onClick={() => remove(q)}>Remove</button>
                </div>
              )}
            </div>
            <p className="qbody">{q.body}</p>
            {q.options?.length > 0 && (
              <ol className="qoptions" type="A">
                {q.options.map((o, idx) => {
                  const letter = 'ABCD'[idx];
                  const right = q.correct === letter;
                  return <li key={idx} className={right ? 'correct' : ''}>{o}{right && <span className="tick"> &#10003; correct</span>}</li>;
                })}
              </ol>
            )}
            {q.rubric && <div className="rubric"><strong>Rubric:</strong> {q.rubric}</div>}
          </div>
        ))
      )}

      {editing && (
        <Modal title="Edit question" onClose={() => setEditing(null)}>
          <QuestionEditor initial={editing} onSave={saveEdit} onCancel={() => setEditing(null)} saving={saving} />
        </Modal>
      )}
      {adding && (
        <Modal title="Add custom question" onClose={() => setAdding(false)}>
          <QuestionEditor onSave={saveNew} onCancel={() => setAdding(false)} saving={saving} />
        </Modal>
      )}
    </>
  );
}


src/pages/ParticipantsTab.jsx
import { useCallback, useEffect, useState } from 'react';
import { useOutletContext } from 'react-router-dom';
import { api } from '../api.js';
import Badge from '../components/Badge.jsx';
import { Empty, ErrorBox, Loading } from '../components/PageState.jsx';
import { useUi } from '../context/UiContext.jsx';

const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

export default function ParticipantsTab() {
  const { assessment } = useOutletContext();
  const { toast, confirm } = useUi();
  const approved = assessment.status === 'approved' || assessment.status === 'open';

  const [rows, setRows] = useState(null);
  const [error, setError] = useState(null);
  const [name, setName] = useState('');
  const [email, setEmail] = useState('');
  const [csv, setCsv] = useState('');
  const [busy, setBusy] = useState(false);

  const load = useCallback(async (silent = false) => {
    try {
      const r = await api.listParticipants(assessment.id);
      setRows(r);
      setError(null);
    } catch (e) { if (!silent) setError(e); }
  }, [assessment.id]);

  useEffect(() => { load(); }, [load]);

  // While invitations are being sent, refresh automatically.
  useEffect(() => {
    if (!rows?.some((r) => r.inviteStatus === 'queued')) return undefined;
    const t = setInterval(() => load(true), 5000);
    return () => clearInterval(t);
  }, [rows, load]);

  const report = (r) => {
    const inv = r.invalid?.length || 0;
    toast(`Added ${r.added}, duplicates ${r.duplicates}, invalid ${inv}.`, r.added > 0 ? 'success' : 'info');
  };

  const addOne = async (e) => {
    e.preventDefault();
    if (!name.trim()) return toast('Enter the participant name.', 'error');
    if (!EMAIL_RE.test(email.trim())) return toast('Enter a valid email address.', 'error');
    setBusy(true);
    try {
      report(await api.addParticipants(assessment.id, { participants: [{ name: name.trim(), email: email.trim() }] }));
      setName(''); setEmail(''); load();
    } catch (err) { toast(err.message, 'error'); }
    finally { setBusy(false); }
  };

  const onCsvFile = (e) => {
    const f = e.target.files?.[0];
    if (!f) return;
    if (f.size > 1024 * 1024) { toast('CSV file is too large.', 'error'); return; }
    const reader = new FileReader();
    reader.onload = () => setCsv(String(reader.result || ''));
    reader.readAsText(f);
    e.target.value = '';
  };

  const addCsv = async () => {
    let text = csv.trim();
    if (!text) return toast('Paste CSV text or choose a file first.', 'error');
    if (!/^\s*name\s*,\s*email/i.test(text)) text = `name,email\n${text}`;
    setBusy(true);
    try {
      report(await api.addParticipants(assessment.id, { csv: text }));
      setCsv(''); load();
    } catch (err) { toast(err.message, 'error'); }
    finally { setBusy(false); }
  };

  const sendAll = async () => {
    const ok = await confirm({
      title: 'Send invitations',
      message: 'Send assessment invitation emails to all participants who have not been invited yet?',
      confirmText: 'Send emails',
    });
    if (!ok) return;
    setBusy(true);
    try {
      const r = await api.sendInvitations(assessment.id, {});
      toast(`Invitations queued${r?.queued !== undefined ? ` (${r.queued})` : ''}. Emails are being sent.`, 'success');
      load();
    } catch (err) {
      toast(err.code === 'NOT_APPROVED' ? 'Approve the questions before sending invitations.' : err.message, 'error');
    } finally { setBusy(false); }
  };

  const resend = async (p) => {
    try { await api.resend(p.id); toast(`New invitation queued for ${p.email}. The old link no longer works.`, 'success'); load(); }
    catch (err) { toast(err.message, 'error'); }
  };

  const remove = async (p) => {
    const ok = await confirm({ title: 'Remove participant', message: `Remove ${p.email}?`, confirmText: 'Remove', danger: true });
    if (!ok) return;
    try { await api.deleteParticipant(p.id); toast('Participant removed.', 'success'); load(); }
    catch (err) { toast(err.message, 'error'); }
  };

  const started = (p) => p.attemptStatus && p.attemptStatus !== 'none';

  return (
    <>
      <div className="grid-2">
        <form className="card" onSubmit={addOne} noValidate>
          <h3 className="card-title">Add a participant</h3>
          <div className="form-group"><label>Name</label><input className="input" value={name} onChange={(e) => setName(e.target.value)} /></div>
          <div className="form-group"><label>Email</label><input className="input" type="email" value={email} onChange={(e) => setEmail(e.target.value)} /></div>
          <button className="btn btn-primary" disabled={busy}>Add participant</button>
        </form>

        <div className="card">
          <h3 className="card-title">Bulk add (CSV)</h3>
          <p className="muted">One participant per line: <code>name,email</code>. Maximum 200 per assessment.</p>
          <textarea className="input" rows={5} value={csv} onChange={(e) => setCsv(e.target.value)} placeholder={'Asha Rao,asha@example.com\nVikram Shah,vikram@example.com'} />
          <div className="toolbar">
            <input type="file" accept=".csv,text/csv,text/plain" onChange={onCsvFile} />
            <span className="grow" />
            <button className="btn btn-primary" onClick={addCsv} disabled={busy}>Add from CSV</button>
          </div>
        </div>
      </div>

      <div className="card">
        <div className="card-head">
          <h3 className="card-title">Participants {rows ? `(${rows.length})` : ''}</h3>
          <div className="toolbar tight">
            <button className="btn btn-ghost btn-sm" onClick={() => load()}>Refresh</button>
            <button className="btn btn-primary" onClick={sendAll} disabled={busy || !approved || !rows?.length}>Send invitations</button>
          </div>
        </div>
        {!approved && <div className="alert alert-info">Approve the questions first. Invitations can only be sent for approved assessments.</div>}

        {error ? <ErrorBox error={error} onRetry={load} />
          : !rows ? <Loading />
          : rows.length === 0 ? <Empty title="No participants yet" text="Add participants above." />
          : (
            <div className="table-wrap">
              <table className="table">
                <thead><tr><th>Name</th><th>Email</th><th>Invitation</th><th>Attempt</th><th>Score</th><th /></tr></thead>
                <tbody>
                  {rows.map((p) => (
                    <tr key={p.id}>
                      <td><strong>{p.name}</strong></td>
                      <td>{p.email}</td>
                      <td><Badge value={p.inviteStatus} /></td>
                      <td><Badge value={p.attemptStatus || 'none'} /></td>
                      <td>{p.percent !== undefined && p.percent !== null ? `${Math.round(p.percent)}%` : '-'}</td>
                      <td className="right">
                        {approved && !started(p) && (p.inviteStatus === 'sent' || p.inviteStatus === 'failed') && (
                          <button className="btn btn-ghost btn-sm" onClick={() => resend(p)}>Resend</button>
                        )}
                        {!started(p) && (
                          <button className="btn btn-ghost btn-sm danger-text" onClick={() => remove(p)}>Remove</button>
                        )}
                      </td>
                    </tr>
                  ))}
                </tbody>
              </table>
            </div>
          )}
      </div>
    </>
  );
}



src/pages/ResultsTab.jsx
import { useCallback, useEffect, useState } from 'react';
import { useNavigate, useOutletContext } from 'react-router-dom';
import { api, downloadExport } from '../api.js';
import Badge from '../components/Badge.jsx';
import { Empty, ErrorBox, Loading } from '../components/PageState.jsx';
import { useUi } from '../context/UiContext.jsx';
import { fmtNum, formatDuration } from '../utils.js';

export default function ResultsTab() {
  const { assessment } = useOutletContext();
  const navigate = useNavigate();
  const { toast } = useUi();
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [sort, setSort] = useState('score');
  const [exporting, setExporting] = useState('');

  const load = useCallback(async () => {
    try { setError(null); setData(await api.results(assessment.id, sort)); }
    catch (e) { setError(e); }
  }, [assessment.id, sort]);

  useEffect(() => { setData(null); load(); }, [load]);

  const exportAs = async (format) => {
    setExporting(format);
    try { await downloadExport(assessment.id, format); toast(`${format.toUpperCase()} downloaded.`, 'success'); }
    catch (e) { toast(e.message, 'error'); }
    finally { setExporting(''); }
  };

  if (error) return <ErrorBox error={error} onRetry={load} />;
  if (!data) return <Loading />;

  const { summary, rows } = data;

  return (
    <>
      <div className="stats">
        <div className="stat"><span>Candidates</span><strong>{summary?.count ?? 0}</strong></div>
        <div className="stat"><span>Average score</span><strong>{summary?.avg !== undefined ? `${fmtNum(summary.avg)}%` : '-'}</strong></div>
        <div className="stat"><span>Pass rate</span><strong>{summary?.passRate !== undefined ? `${fmtNum(summary.passRate <= 1 ? summary.passRate * 100 : summary.passRate, 0)}%` : '-'}</strong></div>
      </div>

      <div className="card">
        <div className="card-head">
          <h3 className="card-title">Candidate results</h3>
          <div className="toolbar tight">
            <select className="input input-auto" value={sort} onChange={(e) => setSort(e.target.value)}>
              <option value="score">Sort by score</option>
              <option value="name">Sort by name</option>
            </select>
            <button className="btn btn-outline btn-sm" onClick={() => exportAs('csv')} disabled={Boolean(exporting)}>{exporting === 'csv' ? 'Preparing...' : 'Export CSV'}</button>
            <button className="btn btn-outline btn-sm" onClick={() => exportAs('pdf')} disabled={Boolean(exporting)}>{exporting === 'pdf' ? 'Preparing...' : 'Export PDF'}</button>
          </div>
        </div>

        {!rows || rows.length === 0 ? (
          <Empty title="No results yet" text="Results appear here once candidates submit their attempts." />
        ) : (
          <div className="table-wrap">
            <table className="table">
              <thead><tr><th>#</th><th>Candidate</th><th>Score</th><th>Percent</th><th>Result</th><th>Time taken</th><th>Status</th></tr></thead>
              <tbody>
                {rows.map((r, i) => (
                  <tr key={r.attemptId} className="clickable" onClick={() => navigate(`/attempts/${r.attemptId}`)}>
                    <td>{i + 1}</td>
                    <td><strong>{r.name}</strong><div className="muted small">{r.email}</div></td>
                    <td>{fmtNum(r.score)} / {fmtNum(r.maxScore)}</td>
                    <td><strong>{fmtNum(r.percent)}%</strong></td>
                    <td><span className={`badge ${r.passed ? 'b-open' : 'b-failed'}`}>{r.passed ? 'Passed' : 'Not passed'}</span></td>
                    <td>{formatDuration(r.timeTakenSec)}</td>
                    <td><Badge value={r.status} /></td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        )}
      </div>
    </>
  );
}



src/pages/AttemptPage.jsx
import { useCallback, useEffect, useState } from 'react';
import { useNavigate, useParams } from 'react-router-dom';
import { api } from '../api.js';
import Badge from '../components/Badge.jsx';
import Modal from '../components/Modal.jsx';
import { ErrorBox, Loading } from '../components/PageState.jsx';
import { useUi } from '../context/UiContext.jsx';
import { TYPE_LABEL, fmtNum } from '../utils.js';

export default function AttemptPage() {
  const { attemptId } = useParams();
  const navigate = useNavigate();
  const { toast } = useUi();
  const [data, setData] = useState(null);
  const [error, setError] = useState(null);
  const [target, setTarget] = useState(null);
  const [score, setScore] = useState('');
  const [comment, setComment] = useState('');
  const [saving, setSaving] = useState(false);

  const load = useCallback(async () => {
    try { setError(null); setData(await api.getAttempt(attemptId)); }
    catch (e) { setError(e); }
  }, [attemptId]);

  useEffect(() => { load(); }, [load]);

  if (error) return <ErrorBox error={error} onRetry={load} />;
  if (!data) return <Loading />;

  const answers = data.answers || [];
  const total = answers.reduce((a, x) => a + (Number(x.score) || 0), 0);
  const max = answers.reduce((a, x) => a + (Number(x.maxScore) || 0), 0);

  const openOverride = (a) => { setTarget(a); setScore(String(a.score ?? 0)); setComment(''); };

  const saveOverride = async () => {
    const s = Number(score);
    if (!Number.isFinite(s) || s < 0 || s > Number(target.maxScore)) {
      return toast(`Score must be between 0 and ${target.maxScore}.`, 'error');
    }
    if (!comment.trim()) return toast('Please add a comment explaining the change.', 'error');
    setSaving(true);
    try {
      await api.override(attemptId, target.questionId, { score: s, comment: comment.trim() });
      toast('Score updated.', 'success');
      setTarget(null);
      load();
    } catch (e) { toast(e.message, 'error'); }
    finally { setSaving(false); }
  };

  return (
    <>
      <button className="link-back" onClick={() => navigate(-1)}>&larr; Back to results</button>
      <div className="page-header">
        <div>
          <h1>{data.participant?.name || 'Candidate'}</h1>
          <p className="muted">{data.participant?.email}</p>
        </div>
        <div className="score-pill">
          <span>Total</span>
          <strong>{fmtNum(total)} / {fmtNum(max)}</strong>
        </div>
      </div>

      {answers.map((a, i) => (
        <div className="card qcard" key={a.questionId}>
          <div className="qhead">
            <div className="qmeta">
              <span className="qnum">Q{i + 1}</span>
              <span className="badge b-approved">{TYPE_LABEL[a.type] || a.type}</span>
              <span className="badge b-none">scored by {a.source}</span>
              {a.type !== 'descriptive' && a.correct !== undefined && (
                <span className={`badge ${String(a.answer) === String(a.correct) ? 'b-open' : 'b-failed'}`}>
                  {String(a.answer) === String(a.correct) ? 'Correct' : 'Incorrect'}
                </span>
              )}
            </div>
            <div className="qactions">
              <strong>{fmtNum(a.score)} / {fmtNum(a.maxScore)}</strong>
              <button className="btn btn-ghost btn-sm" onClick={() => openOverride(a)}>Override</button>
            </div>
          </div>
          <p className="qbody">{a.body}</p>
          <div className="answer-box">
            <div className="label">Candidate answer</div>
            <div>{a.answer ? a.answer : <em className="muted">No answer given</em>}</div>
          </div>
          {a.type !== 'descriptive' && a.correct && <div className="muted small">Correct answer: {a.correct}</div>}
          {a.justification && (
            <div className="answer-box ai">
              <div className="label">{a.source === 'manual' ? 'Reviewer comment' : 'AI justification'}</div>
              <div>{a.justification}</div>
            </div>
          )}
          {a.status === 'pending_review' && <Badge value="needs_review" />}
        </div>
      ))}

      {target && (
        <Modal
          title="Override score"
          onClose={() => setTarget(null)}
          footer={<>
            <button className="btn btn-ghost" onClick={() => setTarget(null)}>Cancel</button>
            <button className="btn btn-primary" onClick={saveOverride} disabled={saving}>{saving ? 'Saving...' : 'Save score'}</button>
          </>}
        >
          <p className="muted">The original AI score is kept in the audit log.</p>
          <div className="form-group">
            <label>New score (0 to {target.maxScore})</label>
            <input className="input" type="number" step="0.5" value={score} onChange={(e) => setScore(e.target.value)} />
          </div>
          <div className="form-group">
            <label>Comment (required)</label>
            <textarea className="input" rows={3} value={comment} onChange={(e) => setComment(e.target.value)} />
          </div>
        </Modal>
      )}
    </>
  );
}




src/pages/UsersPage.jsx
import { useCallback, useEffect, useState } from 'react';
import { api } from '../api.js';
import Badge from '../components/Badge.jsx';
import Modal from '../components/Modal.jsx';
import { ErrorBox, Loading } from '../components/PageState.jsx';
import { useAuth } from '../context/AuthContext.jsx';
import { useUi } from '../context/UiContext.jsx';

const EMAIL_RE = /^[^\s@]+@[^\s@]+\.[^\s@]+$/;

export default function UsersPage() {
  const { user: me } = useAuth();
  const { toast } = useUi();
  const [users, setUsers] = useState(null);
  const [error, setError] = useState(null);
  const [open, setOpen] = useState(false);
  const [form, setForm] = useState({ name: '', email: '', role: 'recruiter', password: '' });
  const [saving, setSaving] = useState(false);

  const load = useCallback(async () => {
    try { setError(null); setUsers(await api.listUsers()); } catch (e) { setError(e); }
  }, []);
  useEffect(() => { load(); }, [load]);

  const create = async () => {
    if (!form.name.trim()) return toast('Name is required.', 'error');
    if (!EMAIL_RE.test(form.email.trim())) return toast('Enter a valid email.', 'error');
    if (form.password.length < 8) return toast('Password must be at least 8 characters.', 'error');
    setSaving(true);
    try {
      await api.createUser({ ...form, name: form.name.trim(), email: form.email.trim() });
      toast('User created.', 'success');
      setOpen(false);
      setForm({ name: '', email: '', role: 'recruiter', password: '' });
      load();
    } catch (e) {
      toast(e.code === 'EMAIL_EXISTS' ? 'A user with this email already exists.' : e.message, 'error');
    } finally { setSaving(false); }
  };

  const patch = async (u, body) => {
    try { await api.updateUser(u.id, body); toast('User updated.', 'success'); load(); }
    catch (e) { toast(e.message, 'error'); }
  };

  return (
    <>
      <div className="page-header">
        <div><h1>Team &amp; Users</h1><p className="muted">Manage who can access your workspace.</p></div>
        <button className="btn btn-primary" onClick={() => setOpen(true)}>+ Add user</button>
      </div>

      <div className="card">
        {error ? <ErrorBox error={error} onRetry={load} /> : !users ? <Loading /> : (
          <div className="table-wrap">
            <table className="table">
              <thead><tr><th>Name</th><th>Email</th><th>Role</th><th>Status</th><th /></tr></thead>
              <tbody>
                {users.map((u) => (
                  <tr key={u.id}>
                    <td><strong>{u.name}</strong>{u.id === me.id && <span className="muted"> (you)</span>}</td>
                    <td>{u.email}</td>
                    <td>
                      <select className="input input-auto" value={u.role} disabled={u.id === me.id} onChange={(e) => patch(u, { role: e.target.value })}>
                        <option value="admin">Admin</option>
                        <option value="recruiter">Recruiter</option>
                      </select>
                    </td>
                    <td><Badge value={u.status} /></td>
                    <td className="right">
                      {u.id !== me.id && (
                        <button className="btn btn-ghost btn-sm" onClick={() => patch(u, { status: u.status === 'active' ? 'disabled' : 'active' })}>
                          {u.status === 'active' ? 'Disable' : 'Enable'}
                        </button>
                      )}
                    </td>
                  </tr>
                ))}
              </tbody>
            </table>
          </div>
        )}
      </div>

      {open && (
        <Modal title="Add user" onClose={() => setOpen(false)}
          footer={<>
            <button className="btn btn-ghost" onClick={() => setOpen(false)}>Cancel</button>
            <button className="btn btn-primary" onClick={create} disabled={saving}>{saving ? 'Creating...' : 'Create user'}</button>
          </>}>
          <div className="form-group"><label>Name</label><input className="input" value={form.name} onChange={(e) => setForm({ ...form, name: e.target.value })} /></div>
          <div className="form-group"><label>Email</label><input className="input" type="email" value={form.email} onChange={(e) => setForm({ ...form, email: e.target.value })} /></div>
          <div className="form-group"><label>Role</label>
            <select className="input" value={form.role} onChange={(e) => setForm({ ...form, role: e.target.value })}>
              <option value="recruiter">Recruiter</option><option value="admin">Admin</option>
            </select>
          </div>
          <div className="form-group"><label>Temporary password</label><input className="input" type="password" autoComplete="new-password" value={form.password} onChange={(e) => setForm({ ...form, password: e.target.value })} /></div>
        </Modal>
      )}
    </>
  );
}


**src/pages/NotFoundPage.jsx**
import { Link } from 'react-router-dom';

export default function NotFoundPage() {
  return (
    <div className="state">
      <h2>Page not found</h2>
      <p className="muted">The page you are looking for does not exist.</p>
      <Link className="btn btn-primary" to="/assessments">Go to assessments</Link>
    </div>
  );
}



**Step 7: Theme: src/styles.css**
:root {
  --navy: #0a1f44;
  --navy-2: #12306b;
  --blue: #1d4ed8;
  --blue-dark: #1e40af;
  --sky: #e8efff;
  --bg: #f3f6fc;
  --card: #ffffff;
  --text: #0f1b33;
  --muted: #5b6b88;
  --border: #d9e2f2;
  --green: #15803d;
  --amber: #b45309;
  --red: #b91c1c;
  --radius: 10px;
  --shadow: 0 1px 3px rgba(10, 31, 68, 0.08), 0 4px 14px rgba(10, 31, 68, 0.05);
}

* { box-sizing: border-box; }
html, body, #root { height: 100%; }
body {
  margin: 0;
  font-family: 'Times New Roman', Times, serif;
  font-size: 17px;
  color: var(--text);
  background: var(--bg);
}
h1, h2, h3 { margin: 0; color: var(--navy); }
h1 { font-size: 30px; }
h2 { font-size: 26px; }
p { margin: 0 0 10px; }
code { background: var(--sky); padding: 1px 6px; border-radius: 4px; font-family: 'Courier New', monospace; font-size: 14px; }
.muted { color: var(--muted); }
.small { font-size: 14px; }
.right { text-align: right; white-space: nowrap; }
.grow { flex: 1; }
.danger-text { color: var(--red); }

/* Layout shell */
.shell { display: flex; min-height: 100%; }
.sidebar { width: 250px; background: linear-gradient(180deg, var(--navy), var(--navy-2)); color: #fff; display: flex; flex-direction: column; position: sticky; top: 0; height: 100vh; }
.brand { display: flex; align-items: center; gap: 12px; padding: 24px 20px; }
.brand-mark { width: 42px; height: 42px; border-radius: 10px; background: #fff; color: var(--navy); display: flex; align-items: center; justify-content: center; font-weight: bold; font-size: 24px; }
.brand-mark-lg { width: 64px; height: 64px; font-size: 36px; border-radius: 14px; margin-bottom: 18px; }
.brand-name { font-size: 22px; font-weight: bold; }
.brand-sub { font-size: 13px; opacity: 0.75; }
.nav { display: flex; flex-direction: column; padding: 8px 12px; gap: 4px; }
.nav a { color: #cfdcff; text-decoration: none; padding: 11px 14px; border-radius: 8px; font-size: 18px; }
.nav a:hover { background: rgba(255, 255, 255, 0.08); }
.nav a.active { background: #fff; color: var(--navy); font-weight: bold; }
.sidebar-foot { margin-top: auto; padding: 18px 20px; font-size: 14px; opacity: 0.7; border-top: 1px solid rgba(255, 255, 255, 0.12); }
.main { flex: 1; min-width: 0; display: flex; flex-direction: column; }
.topbar { background: #fff; border-bottom: 1px solid var(--border); display: flex; justify-content: space-between; align-items: center; padding: 12px 32px; }
.topbar-user { display: flex; align-items: center; gap: 16px; }
.role-chip { margin-left: 10px; background: var(--sky); color: var(--blue-dark); padding: 2px 10px; border-radius: 999px; font-size: 13px; text-transform: capitalize; }
.content { padding: 28px 32px 60px; max-width: 1280px; width: 100%; }

/* Page bits */
.page-header { display: flex; justify-content: space-between; align-items: flex-start; gap: 16px; margin-bottom: 22px; flex-wrap: wrap; }
.meta-line { display: flex; align-items: center; gap: 12px; margin-top: 8px; }
.link-back { background: none; border: 0; color: var(--blue); cursor: pointer; font: inherit; padding: 0; margin-bottom: 12px; }
.link-back:hover { text-decoration: underline; }
.tabs { display: flex; gap: 6px; border-bottom: 2px solid var(--border); margin-bottom: 22px; flex-wrap: wrap; }
.tab { padding: 10px 18px; text-decoration: none; color: var(--muted); font-size: 18px; border-bottom: 3px solid transparent; margin-bottom: -2px; }
.tab:hover { color: var(--navy); }
.tab.active { color: var(--blue); border-bottom-color: var(--blue); font-weight: bold; }

.card { background: var(--card); border: 1px solid var(--border); border-radius: var(--radius); box-shadow: var(--shadow); padding: 22px; margin-bottom: 20px; }
.card-head { display: flex; justify-content: space-between; align-items: center; gap: 12px; margin-bottom: 16px; flex-wrap: wrap; }
.card-title { font-size: 21px; margin-bottom: 10px; }
.card-head .card-title { margin-bottom: 0; }
.grid-2 { display: grid; grid-template-columns: 1fr 1fr; gap: 20px; }
.stats { display: grid; grid-template-columns: repeat(auto-fit, minmax(190px, 1fr)); gap: 16px; margin-bottom: 20px; }
.stat { background: #fff; border: 1px solid var(--border); border-left: 5px solid var(--blue); border-radius: var(--radius); padding: 16px 20px; box-shadow: var(--shadow); }
.stat span { display: block; color: var(--muted); font-size: 15px; }
.stat strong { font-size: 32px; color: var(--navy); }
.score-pill { background: var(--navy); color: #fff; padding: 12px 22px; border-radius: var(--radius); text-align: center; }
.score-pill span { display: block; font-size: 13px; opacity: 0.8; }
.score-pill strong { font-size: 26px; }
.details { display: grid; grid-template-columns: 170px 1fr; gap: 10px 16px; margin: 14px 0; }
.details dt { color: var(--muted); }
.details dd { margin: 0; }
.jd-box { white-space: pre-wrap; background: var(--bg); border: 1px solid var(--border); border-radius: 8px; padding: 14px; font-family: inherit; font-size: 15px; max-height: 320px; overflow: auto; }

/* Buttons */
.btn { font-family: inherit; font-size: 17px; border-radius: 8px; border: 1px solid transparent; padding: 10px 20px; cursor: pointer; text-decoration: none; display: inline-flex; align-items: center; justify-content: center; gap: 8px; transition: background 0.15s, box-shadow 0.15s; }
.btn:disabled { opacity: 0.55; cursor: not-allowed; }
.btn-sm { padding: 6px 12px; font-size: 15px; }
.btn-block { width: 100%; }
.btn-primary { background: var(--blue); color: #fff; }
.btn-primary:hover:not(:disabled) { background: var(--blue-dark); }
.btn-success { background: var(--green); color: #fff; }
.btn-danger { background: var(--red); color: #fff; }
.btn-outline { background: #fff; color: var(--blue); border-color: var(--blue); }
.btn-outline:hover:not(:disabled) { background: var(--sky); }
.btn-ghost { background: transparent; color: var(--blue-dark); }
.btn-ghost:hover:not(:disabled) { background: var(--sky); }
.icon-btn { background: none; border: 0; font-size: 28px; line-height: 1; cursor: pointer; color: var(--muted); }
.toolbar { display: flex; gap: 12px; align-items: center; margin: 0 0 18px; flex-wrap: wrap; }
.toolbar.tight { margin: 0; }
.form-actions { display: flex; justify-content: flex-end; margin-bottom: 40px; }

/* Forms */
.form-group { margin-bottom: 16px; }
.form-group label { display: block; margin-bottom: 6px; font-weight: bold; font-size: 16px; color: var(--navy); }
.form-row { display: grid; grid-template-columns: 1fr 1fr; gap: 16px; }
.input { width: 100%; font-family: inherit; font-size: 17px; padding: 10px 12px; border: 1px solid var(--border); border-radius: 8px; background: #fff; color: var(--text); }
.input:focus { outline: 2px solid var(--blue); border-color: var(--blue); }
.input-auto { width: auto; }
textarea.input { resize: vertical; }
.field-error { color: var(--red); font-size: 14px; margin-top: 4px; }
.hint { font-size: 14px; color: var(--muted); margin-top: 4px; }
.hint-warn { color: var(--amber); }
.section-row { display: flex; gap: 14px; align-items: flex-start; padding: 14px; background: var(--bg); border-radius: 8px; margin-bottom: 12px; flex-wrap: wrap; }
.section-row .form-group { margin-bottom: 0; }
.section-row .grow { flex: 1; min-width: 160px; }
.section-row .narrow { width: 90px; }
.remove-btn { margin-top: 28px; }

/* Table */
.table-wrap { overflow-x: auto; }
.table { width: 100%; border-collapse: collapse; }
.table th { text-align: left; font-size: 14px; text-transform: uppercase; letter-spacing: 0.04em; color: var(--muted); padding: 10px 12px; border-bottom: 2px solid var(--border); white-space: nowrap; }
.table td { padding: 12px; border-bottom: 1px solid var(--border); vertical-align: middle; }
.table tr.clickable { cursor: pointer; }
.table tr.clickable:hover { background: var(--sky); }

/* Badges */
.badge { display: inline-block; padding: 3px 11px; border-radius: 999px; font-size: 13px; text-transform: capitalize; background: #e5e9f2; color: #3b4863; white-space: nowrap; }
.b-approved, .b-queued, .b-in_progress, .b-submitted { background: #dbe7ff; color: var(--blue-dark); }
.b-open, .b-sent, .b-scored, .b-active { background: #dcfce7; color: var(--green); }
.b-closed { background: var(--navy); color: #fff; }
.b-failed, .b-disabled { background: #fee2e2; color: var(--red); }
.b-needs_review { background: #fef3c7; color: var(--amber); }

/* Questions */
.qcard { padding: 18px 22px; }
.qhead { display: flex; justify-content: space-between; align-items: center; gap: 12px; flex-wrap: wrap; margin-bottom: 10px; }
.qmeta, .qactions { display: flex; align-items: center; gap: 10px; flex-wrap: wrap; }
.qnum { font-weight: bold; color: var(--blue); font-size: 19px; }
.qbody { font-size: 18px; margin: 6px 0 10px; }
.qoptions { margin: 0 0 10px; padding-left: 28px; }
.qoptions li { padding: 4px 0; }
.qoptions li.correct { color: var(--green); font-weight: bold; }
.tick { font-size: 14px; }
.rubric { background: var(--bg); border-left: 4px solid var(--blue); padding: 10px 14px; border-radius: 6px; font-size: 15px; }
.answer-box { background: var(--bg); border: 1px solid var(--border); border-radius: 8px; padding: 12px 14px; margin: 10px 0; white-space: pre-wrap; }
.answer-box.ai { background: var(--sky); border-color: #c4d4ff; }
.answer-box .label { font-size: 13px; text-transform: uppercase; letter-spacing: 0.05em; color: var(--muted); margin-bottom: 4px; }

/* Alerts, states */
.alert { display: flex; align-items: center; gap: 10px; padding: 12px 16px; border-radius: 8px; margin-bottom: 16px; }
.alert-info { background: var(--sky); color: var(--blue-dark); border: 1px solid #c4d4ff; }
.alert-error { background: #fee2e2; color: var(--red); border: 1px solid #fecaca; }
.state { text-align: center; padding: 48px 16px; color: var(--muted); }
.state-full { min-height: 100vh; display: flex; flex-direction: column; align-items: center; justify-content: center; }
.state-icon { width: 48px; height: 48px; border-radius: 50%; background: #fee2e2; color: var(--red); font-size: 26px; font-weight: bold; display: inline-flex; align-items: center; justify-content: center; margin-bottom: 10px; }
.state-icon-soft { background: var(--sky); color: var(--blue); }
.spinner { width: 34px; height: 34px; border: 4px solid var(--sky); border-top-color: var(--blue); border-radius: 50%; animation: spin 0.8s linear infinite; margin: 0 auto 12px; }
.spinner-sm { width: 18px; height: 18px; border-width: 3px; margin: 0; flex: none; }
@keyframes spin { to { transform: rotate(360deg); } }

/* Modal + toasts */
.modal-overlay { position: fixed; inset: 0; background: rgba(10, 31, 68, 0.55); display: flex; align-items: flex-start; justify-content: center; padding: 6vh 16px; overflow-y: auto; z-index: 100; }
.modal { background: #fff; border-radius: 12px; width: 100%; box-shadow: 0 20px 50px rgba(0, 0, 0, 0.3); }
.modal-sm { max-width: 440px; }
.modal-md { max-width: 640px; }
.modal-head { display: flex; justify-content: space-between; align-items: center; padding: 18px 22px; border-bottom: 1px solid var(--border); }
.modal-body { padding: 22px; }
.modal-foot, .modal-actions { display: flex; justify-content: flex-end; gap: 10px; }
.modal-foot { padding: 14px 22px; border-top: 1px solid var(--border); }
.modal-actions { margin-top: 12px; }
.toasts { position: fixed; right: 20px; bottom: 20px; display: flex; flex-direction: column; gap: 10px; z-index: 200; max-width: 380px; }
.toast { padding: 12px 16px; border-radius: 8px; color: #fff; background: var(--navy); box-shadow: var(--shadow); font-size: 16px; }
.toast-success { background: var(--green); }
.toast-error { background: var(--red); }

/* Login */
.login-wrap { display: flex; min-height: 100vh; }
.login-hero { flex: 1; background: linear-gradient(145deg, var(--navy), var(--navy-2) 70%, #1d4ed8); color: #fff; padding: 80px 64px; display: flex; flex-direction: column; justify-content: center; }
.login-hero h1 { color: #fff; font-size: 46px; margin-bottom: 12px; }
.login-hero p { font-size: 20px; max-width: 460px; opacity: 0.9; line-height: 1.5; }
.login-panel { flex: 1; display: flex; align-items: center; justify-content: center; padding: 32px; background: #fff; }
.login-card { width: 100%; max-width: 400px; }
.login-card h2 { margin-bottom: 4px; }
.login-card .muted { margin-bottom: 22px; }

@media (max-width: 900px) {
  .sidebar { display: none; }
  .grid-2, .form-row { grid-template-columns: 1fr; }
  .login-hero { display: none; }
  .content { padding: 20px 16px 50px; }
  .topbar { padding: 10px 16px; }
  .details { grid-template-columns: 1fr; }
}
