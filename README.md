# ============================================================
# DIGITAL FOOD SAFETY & HACCP GOVERNANCE PLATFORM
# SINGLE CONSOLIDATED APP.PY (FINAL MVP)
# ============================================================

import streamlit as st
import pandas as pd
from datetime import datetime, timedelta
import hashlib
import io
import json
import os
from fpdf import FPDF

# ------------------------------------------------------------
# PAGE CONFIG
# ------------------------------------------------------------
st.set_page_config(page_title="Digital Food Safety System", layout="wide")

# ------------------------------------------------------------
# BASIC CSS (Mobile + Desktop)
# ------------------------------------------------------------
st.markdown("""
<style>
body { font-size:16px; }
.stButton>button { width:100%; padding:12px; border-radius:10px; }
.card { padding:14px; border-radius:12px; margin-bottom:10px;
        box-shadow:0 4px 10px rgba(0,0,0,0.1); }
.bottom-nav {
 position:fixed; bottom:0; left:0; right:0;
 height:65px; background:#111;
 display:flex; justify-content:space-around;
 align-items:center; z-index:999;
}
.nav-item { color:#ccc; text-align:center;
 font-size:12px; text-decoration:none; }
.nav-item span { font-size:18px; display:block; }
body { padding-bottom:80px; }
</style>
""", unsafe_allow_html=True)

# ------------------------------------------------------------
# SESSION STATE
# ------------------------------------------------------------
for k in [
    "user","role","branch",
    "audit_locked","signed_auditor","signed_manager",
    "theme","is_mobile"
]:
    if k not in st.session_state:
        st.session_state[k] = None

# ------------------------------------------------------------
# USER MASTER (RBAC)
# ------------------------------------------------------------
users_df = pd.DataFrame([
 {"email":"admin@sys.com","role":"Admin","branch":None},
 {"email":"auditor@sys.com","role":"Auditor","branch":None},
 {"email":"manager_cbe@sys.com","role":"Manager","branch":"CBE01"}
])

# ------------------------------------------------------------
# BRANCH MASTER
# ------------------------------------------------------------
branches_df = pd.DataFrame([
 {"Branch Code":"CBE01","Branch Name":"Coimbatore Central"},
 {"Branch Code":"CBE02","Branch Name":"Coimbatore North"},
 {"Branch Code":"CBE03","Branch Name":"Coimbatore South"}
])

# ------------------------------------------------------------
# DOCUMENT MASTER
# ------------------------------------------------------------
document_master = pd.DataFrame([
 {"Doc":"FSSAI License","Mandatory":True,"Validity":True},
 {"Doc":"Fire Extinguisher Cert","Mandatory":True,"Validity":True},
 {"Doc":"Food Handler Medical","Mandatory":True,"Validity":True},
 {"Doc":"Attendance Register","Mandatory":True,"Validity":False},
 {"Doc":"Hygiene Log","Mandatory":True,"Validity":False},
 {"Doc":"Pest Control Report","Mandatory":True,"Validity":True}
])

# ------------------------------------------------------------
# DATA TABLES (IN-MEMORY)
# ------------------------------------------------------------
documents_df = pd.DataFrame(columns=[
 "Branch","Doc","Upload Date","Expiry","Status"
])

daily_logs_df = pd.DataFrame(columns=[
 "Branch","Log Type","Date","Late"
])

haccp_df = pd.DataFrame(columns=[
 "Process Step","Hazard","Description",
 "Control","CCP","Critical Limit","Monitoring","Responsible"
])

ccp_monitor_df = pd.DataFrame(columns=[
 "Process Step","Observed","Limit","Status","Date"
])

audit_df = pd.DataFrame(columns=[
 "Section","Question","Score","Locked"
])

capa_df = pd.DataFrame(columns=[
 "Issue","Severity","Action",
 "Responsible","Deadline","Status","Risk"
])

signature_df = pd.DataFrame(columns=[
 "Role","User","Timestamp","Hash"
])

# ------------------------------------------------------------
# LOGIN
# ------------------------------------------------------------
st.sidebar.markdown("## 🔐 Login")
email = st.sidebar.text_input("Email")
if st.sidebar.button("Login"):
    u = users_df[users_df["email"]==email]
    if not u.empty:
        st.session_state.user = email
        st.session_state.role = u.iloc[0]["role"]
        st.session_state.branch = u.iloc[0]["branch"]
        st.sidebar.success(f"Logged in as {st.session_state.role}")
    else:
        st.sidebar.error("Unauthorized")

if not st.session_state.user:
    st.stop()

role = st.session_state.role
branch = st.session_state.branch

# ------------------------------------------------------------
# THEME TOGGLE
# ------------------------------------------------------------
st.sidebar.markdown("## 🎨 Theme")
theme = st.sidebar.radio("Theme",["Light","Dark"])
st.session_state.theme = theme

if theme=="Dark":
    st.markdown("<style>body{background:#0e1117;color:#fafafa;}</style>",
                unsafe_allow_html=True)

# ------------------------------------------------------------
# UTILITIES
# ------------------------------------------------------------
def compliance_label(score):
    if score>=90: return "🟢 Excellent"
    if score>=75: return "🟡 Good"
    if score>=60: return "🟠 Warning"
    return "🔴 Critical"

def calculate_compliance(branch):
    docs = documents_df[documents_df["Branch"]==branch]
    logs = daily_logs_df[daily_logs_df["Branch"]==branch]
    mandatory = document_master[document_master["Mandatory"]==True]

    doc_score = (len(docs)/max(len(mandatory),1))*40
    valid = docs[docs["Status"]=="Valid"]
    expiring = docs[docs["Status"]=="Expiring"]
    validity = ((len(valid)+0.5*len(expiring))/max(len(docs),1))*30
    log_score = (len(logs)/300)*20
    ontime = logs[logs["Late"]==False]
    time_score = (len(ontime)/max(len(logs),1))*10

    return round(doc_score+validity+log_score+time_score,2)

# ------------------------------------------------------------
# MOBILE BOTTOM NAV
# ------------------------------------------------------------
def mobile_nav(pages,icons):
    html="<div class='bottom-nav'>"
    for p,i in zip(pages,icons):
        html+=f"<a class='nav-item' href='?page={p}'><span>{i}</span>{p}</a>"
    html+="</div>"
    st.markdown(html,unsafe_allow_html=True)

# ------------------------------------------------------------
# PAGE ROUTING
# ------------------------------------------------------------
page = st.query_params.get("page",["Home"])[0]

# ------------------------------------------------------------
# HEADER
# ------------------------------------------------------------
st.title("🛡️ Digital Food Safety & HACCP Platform")

# ============================================================
# HACCP MODULE
# ============================================================
if page=="HACCP" and role=="Auditor" and not st.session_state.audit_locked:
    st.subheader("🧪 HACCP Setup")
    with st.form("haccp"):
        step = st.text_input("Process Step")
        hazard = st.selectbox("Hazard",
                              ["Biological","Chemical","Physical","Allergen"])
        desc = st.text_area("Hazard Description")
        control = st.text_input("Preventive Measure")
        ccp = st.selectbox("Is CCP?",["No","Yes"])
        limit = st.text_input("Critical Limit")
        monitor = st.text_input("Monitoring Method")
        resp = st.text_input("Responsible")
        if st.form_submit_button("Save HACCP"):
            haccp_df.loc[len(haccp_df)] = [
                step,hazard,desc,control,ccp,limit,monitor,resp
            ]
            st.success("HACCP added")

st.subheader("📋 HACCP Plan")
st.dataframe(haccp_df)

# ============================================================
# CCP MONITORING
# ============================================================
if page=="CCP":
    st.subheader("🌡️ CCP Monitoring")
    ccp_steps = haccp_df[haccp_df["CCP"]=="Yes"]["Process Step"]
    if not ccp_steps.empty:
        step = st.selectbox("CCP Step",ccp_steps)
        obs = st.text_input("Observed Value")
        limit = haccp_df[haccp_df["Process Step"]==step]["Critical Limit"].values[0]
        if st.button("Submit CCP"):
            status="OK"
            if "≥" in limit:
                try:
                    if int(obs.replace("°C","")) < int(limit.replace("≥","").replace("°C","")):
                        status="Deviation"
                        capa_df.loc[len(capa_df)] = [
                            f"CCP deviation at {step}",
                            "Critical",
                            "Immediate corrective action",
                            "Production Incharge",
                            datetime.now()+timedelta(days=1),
                            "Open",
                            5
                        ]
                        st.error("🔴 CCP DEVIATION – CAPA GENERATED")
                except:
                    pass
            ccp_monitor_df.loc[len(ccp_monitor_df)] = [
                step,obs,limit,status,datetime.now()
            ]

st.subheader("📊 CCP Status")
if not ccp_monitor_df.empty:
    st.bar_chart(ccp_monitor_df["Status"].value_counts())

# ============================================================
# BRANCH DASHBOARD
# ============================================================
if page=="Home":
    if role=="Manager":
        score = calculate_compliance(branch)
        st.metric("Compliance Score",f"{score}/100")
        st.progress(score/100)
        st.write(compliance_label(score))

# ============================================================
# INCENTIVES
# ============================================================
st.subheader("🏆 Incentives & Ranking")
scores=[]
for b in branches_df["Branch Code"]:
    scores.append({"Branch":b,"Score":calculate_compliance(b)})
rank_df = pd.DataFrame(scores).sort_values("Score",ascending=False)
rank_df["Rank"]=rank_df.index+1
rank_df["Reward"]=rank_df["Rank"].apply(
 lambda r:"₹50,000 / Trip" if r<=2 else "—"
)
st.dataframe(rank_df)

# ============================================================
# DIGITAL SIGNATURE
# ============================================================
st.subheader("✍️ Digital Signature")

if role=="Auditor" and not st.session_state.signed_auditor:
    st.camera_input("Auditor Selfie")
    if st.button("Sign as Auditor"):
        h = hashlib.sha256(f"{email}{datetime.now()}".encode()).hexdigest()
        signature_df.loc[len(signature_df)] = [
            "Auditor",email,datetime.now(),h
        ]
        st.session_state.signed_auditor=True
        st.success("Auditor signed")

if role=="Manager" and not st.session_state.signed_manager:
    st.camera_input("Manager Selfie")
    if st.button("Sign as Manager"):
        h = hashlib.sha256(f"{email}{datetime.now()}".encode()).hexdigest()
        signature_df.loc[len(signature_df)] = [
            "Manager",email,datetime.now(),h
        ]
        st.session_state.signed_manager=True
        st.success("Manager signed")

if st.session_state.signed_auditor and st.session_state.signed_manager:
    st.session_state.audit_locked=True
    st.success("🔒 AUDIT LOCKED")

# ============================================================
# MOBILE NAV
# ============================================================
if role=="Manager":
    mobile_nav(["Home","Docs","Logs","CAPA","Sign"],["🏠","📁","📋","⚠️","✍️"])
elif role=="Auditor":
    mobile_nav(["Home","HACCP","CCP","CAPA","Sign"],["🏠","🧪","🌡️","⚠️","✍️"])
