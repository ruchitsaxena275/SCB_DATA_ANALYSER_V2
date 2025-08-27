import streamlit as st
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import plotly.express as px
import io

# -------- Constants --------
MODULE_POWER_WP = 545.0
MODULE_VOC = 49.91
VMP_VOC_RATIO = 0.82
NUM_STRINGS = 18

VMP = MODULE_VOC * VMP_VOC_RATIO
I_MODULE_STC = MODULE_POWER_WP / VMP   # ≈ 13.3166 A

CR_LOW_THRESHOLD = 0.90

# -------- Functions --------
def process_file(df):
    # Assume: Col A = timestamp, Col B-S = strings, Col Y = irradiation
    df = df.copy()
    df.iloc[:,0] = pd.to_datetime(df.iloc[:,0])   # timestamp
    df = df.set_index(df.columns[0])

    measured_cols = df.columns[0:NUM_STRINGS]     # B..S
    irr_col = df.columns[23]                      # Y

    measured = df[measured_cols].astype(float)
    irr = df[irr_col].astype(float)

    expected_str_current = I_MODULE_STC * (irr / 1000.0)

    result = measured.copy()
    for i,col in enumerate(measured_cols, start=1):
        result[f"Expected_String_{i}"] = expected_str_current
        result[f"CR_String_{i}"] = np.where(expected_str_current>0,
                                            result[col]/expected_str_current,
                                            np.nan)

    result["Expected_SCB_Current"] = expected_str_current * NUM_STRINGS
    result["Measured_SCB_Current"] = measured.sum(axis=1)
    result["Irradiance_Wm2"] = irr
    return result

def plot_heatmap(df):
    cr_cols = [c for c in df.columns if c.startswith("CR_String_")]
    cr_matrix = df[cr_cols]

    fig, ax = plt.subplots(figsize=(14,6))
    im = ax.imshow(cr_matrix.T, aspect='auto', origin='lower', vmin=0.0, vmax=1.4)
    ax.set_yticks(np.arange(NUM_STRINGS))
    ax.set_yticklabels([f"String {i+1}" for i in range(NUM_STRINGS)])
    xticks = np.linspace(0, len(cr_matrix)-1, min(12, len(cr_matrix))).astype(int)
    ax.set_xticks(xticks)
    ax.set_xticklabels([df.index[i].strftime("%m-%d %H:%M") for i in xticks], rotation=45, ha='right')
    ax.set_title("String Current Ratio Heatmap")
    fig.colorbar(im, ax=ax, label="CR (Measured/Expected)")
    st.pyplot(fig)

def daily_summary(df):
    cr_cols = [c for c in df.columns if c.startswith("CR_String_")]
    df2 = df[cr_cols].copy()
    df2["date"] = df.index.date
    grouped = df2.groupby("date")

    rows = []
    for date, grp in grouped:
        weak = []
        for i,c in enumerate(cr_cols, start=1):
            frac = (grp[c]<CR_LOW_THRESHOLD).mean()
            if frac > 0.3:   # 30% of the time weak
                weak.append(f"String {i}")
        rows.append({"date": date, "weak_strings": ", ".join(weak)})
    return pd.DataFrame(rows)

# -------- Streamlit UI --------
st.title("SCB String Current Analysis Tool")

file = st.file_uploader("Upload Excel file (with fixed format)", type=["xlsx"])
if file:
    df = pd.read_excel(file, engine="openpyxl")
    result = process_file(df)

    st.subheader("Preview of Processed Data")
    st.dataframe(result.head(20))

    st.subheader("Heatmap of Current Ratio (CR)")
    plot_heatmap(result)

    st.subheader("Daily Summary of Weak Strings")
    summary = daily_summary(result)
    st.dataframe(summary)

    # Download buttons
    csv = result.reset_index().to_csv(index=False).encode("utf-8")
    st.download_button("Download Processed CSV", csv, "processed_data.csv", "text/csv")

    csv2 = summary.to_csv(index=False).encode("utf-8")
    st.download_button("Download Daily Summary", csv2, "daily_summary.csv", "text/csv")
