import streamlit as st
import pandas as pd
import numpy as np

# -------------------- PAGE CONFIG --------------------
st.set_page_config(
    page_title="SCB Current Analyzer",
    layout="wide"
)

st.title("🔌 SCB Current Analyzer – ITC Wise")

# -------------------- STATIC MASTER DATA --------------------
# 👉 Replace / extend this using your reference Excel
MASTER_DATA = pd.DataFrame([
    {"ITC": "ITC-01", "Inverter": "INV-01", "SCB_ID": "SCB-01", "Strings": 18},
    {"ITC": "ITC-01", "Inverter": "INV-01", "SCB_ID": "SCB-02", "Strings": 18},
    {"ITC": "ITC-01", "Inverter": "INV-02", "SCB_ID": "SCB-03", "Strings": 20},
    {"ITC": "ITC-02", "Inverter": "INV-01", "SCB_ID": "SCB-01", "Strings": 18},
    {"ITC": "ITC-02", "Inverter": "INV-01", "SCB_ID": "SCB-02", "Strings": 18},
])

# -------------------- SIDEBAR --------------------
st.sidebar.header("⚙️ Configuration")

# ITC Selection
selected_itc = st.sidebar.selectbox(
    "Select ITC",
    sorted(MASTER_DATA["ITC"].unique())
)

# Filter master data for selected ITC
itc_master = MASTER_DATA[MASTER_DATA["ITC"] == selected_itc]

st.sidebar.markdown("### 📂 Upload SCB Current Data")
uploaded_file = st.sidebar.file_uploader(
    "Upload Excel file",
    type=["xlsx"]
)

# -------------------- MAIN LOGIC --------------------
if uploaded_file:
    df_raw = pd.read_excel(uploaded_file)

    st.subheader("🧭 Column Mapping")

    col1, col2 = st.columns(2)

    with col1:
        scb_col = st.selectbox(
            "Select SCB ID column",
            df_raw.columns
        )

    with col2:
        current_col = st.selectbox(
            "Select ISC Current column",
            df_raw.columns
        )

    if st.button("▶ Analyze SCBs"):
        # -------------------- DATA MERGE --------------------
        df = df_raw[[scb_col, current_col]].copy()
        df.columns = ["SCB_ID", "ISC"]

        df["ISC"] = pd.to_numeric(df["ISC"], errors="coerce")

        # Merge with static master
        df = pd.merge(
            itc_master,
            df,
            on="SCB_ID",
            how="left"
        )

        # -------------------- CALCULATIONS --------------------
        df["I_norm"] = df["ISC"] / df["Strings"]

        # Peer Median (ITC + Inverter)
        df["Peer_Median"] = df.groupby("Inverter")["I_norm"].transform("median")

        # Deviation %
        df["Deviation_%"] = ((df["I_norm"] - df["Peer_Median"]) /
                              df["Peer_Median"]) * 100

        # Status Logic
        def status_logic(x):
            if pd.isna(x):
                return "No Data"
            elif x >= -2:
                return "Normal"
            elif x >= -5:
                return "Watch"
            elif x >= -10:
                return "Underperforming"
            else:
                return "Critical"

        df["Status"] = df["Deviation_%"].apply(status_logic)

        # Rank inside inverter
        df["Rank_in_Inverter"] = df.groupby("Inverter")["Deviation_%"] \
                                     .rank(method="dense")

        # Fault Hint
        def fault_hint(row):
            if pd.isna(row["ISC"]):
                return "No current data"
            if row["ISC"] < 0.5:
                return "Fuse blown / string open"
            if row["Deviation_%"] < -10:
                return "Multiple strings / diode / PID"
            if row["Deviation_%"] < -5:
                return "Single string / connector issue"
            return "Healthy"

        df["Probable_Fault"] = df.apply(fault_hint, axis=1)

        # -------------------- KPIs --------------------
        st.subheader("📊 ITC Health Summary")

        k1, k2, k3, k4 = st.columns(4)

        k1.metric("Total SCBs", len(df))
        k2.metric("Critical", (df["Status"] == "Critical").sum())
        k3.metric("Underperforming", (df["Status"] == "Underperforming").sum())
        k4.metric("No Data", (df["Status"] == "No Data").sum())

        # -------------------- TABLE --------------------
        st.subheader("📋 SCB Performance Table")

        styled_df = df.sort_values("Deviation_%").style.applymap(
            lambda v: "background-color:#ff4d4d" if v == "Critical"
            else "background-color:#ffa64d" if v == "Underperforming"
            else "background-color:#fff3cd" if v == "Watch"
            else "background-color:#d4edda" if v == "Normal"
            else "",
            subset=["Status"]
        )

        st.dataframe(styled_df, use_container_width=True)

        # -------------------- CHART --------------------
        st.subheader("📉 Deviation Distribution")

        st.bar_chart(
            df["Deviation_%"].dropna(),
            use_container_width=True
        )

        # -------------------- DOWNLOAD --------------------
        st.subheader("⬇ Download Results")

        output = df.copy()
        st.download_button(
            label="Download SCB Analysis Excel",
            data=output.to_excel(index=False),
            file_name=f"{selected_itc}_SCB_Analysis.xlsx"
        )

else:
    st.info("⬅ Select ITC and upload SCB current data to begin analysis")
