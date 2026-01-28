import streamlit as st
import pandas as pd

st.title("🛡️ HACCP Analysis App (with Pandas)")

Create empty DataFrame
if "haccp" not in st.session_state:
st.session_state.haccp = pd.DataFrame(
columns=[
"Process Step",
"Hazard Type",
"Hazard Description",
"CCP",
"Critical Limit",
"Monitoring",
"Corrective Action"
]
)

Input Form
with st.form("haccp_form"):
step = st.text_input("Process Step")
hazard = st.selectbox("Hazard Type", ["Biological", "Chemical", "Physical"])
desc = st.text_input("Hazard Description")
ccp = st.selectbox("Is it a CCP?", ["Yes", "No"])
limit = st.text_input("Critical Limit")
monitoring = st.text_input("Monitoring Method")
action = st.text_input("Corrective Action")

submit = st.form_submit_button("Add Record")
Add data to Pandas
if submit:
new_row = {
"Process Step": step,
"Hazard Type": hazard,
"Hazard Description": desc,
"CCP": ccp,
"Critical Limit": limit,
"Monitoring": monitoring,
"Corrective Action": action
}

st.session_state.haccp = pd.concat(
    [st.session_state.haccp, pd.DataFrame([new_row])],
    ignore_index=True
)
st.success("HACCP record added ✅")
Display DataFrame
st.subheader("📋 HACCP Table")
st.dataframe(st.session_state.haccp)

