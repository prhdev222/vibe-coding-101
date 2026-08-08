import sqlite3
import statistics
from typing import Dict, List, Optional, Tuple

import streamlit as st


# =====================================================================
# 1. DATA ACCESS LAYER (SQLITE MODEL)
# =====================================================================
class PatientModel:
    """Manages persistent patient data using SQLite."""

    def __init__(self, db_path: str = "diabetes_patients.db"):
        self.db_path = db_path
        self._initialize_database()
        self._seed_initial_data()
        self._clean_initial_data()

    def _get_connection(self):
        return sqlite3.connect(self.db_path)

    def _initialize_database(self) -> None:
        with self._get_connection() as conn:
            conn.execute(
                """
                CREATE TABLE IF NOT EXISTS patients (
                    patient_id INTEGER PRIMARY KEY,
                    Glucose REAL NOT NULL,
                    BMI REAL NOT NULL,
                    Age REAL NOT NULL,
                    BloodPressure REAL NOT NULL
                )
                """
            )
            conn.commit()

    def _seed_initial_data(self) -> None:
        initial_patients = [
            (101, 95.0, 22.5, 28.0, 115.0),
            (102, 145.0, 0.0, 54.0, 135.0),
            (103, 112.0, 29.1, 42.0, 122.0),
            (104, 180.0, 36.4, 61.0, 142.0),
        ]

        with self._get_connection() as conn:
            conn.executemany(
                """
                INSERT OR IGNORE INTO patients
                    (patient_id, Glucose, BMI, Age, BloodPressure)
                VALUES (?, ?, ?, ?, ?)
                """,
                initial_patients,
            )
            conn.commit()

    def _clean_initial_data(self) -> None:
        with self._get_connection() as conn:
            rows = conn.execute(
                """
                SELECT BMI
                FROM patients
                WHERE BMI > 0
                ORDER BY BMI
                """
            ).fetchall()

            valid_bmis = [row[0] for row in rows]
            median_bmi = statistics.median(valid_bmis) if valid_bmis else 25.0

            conn.execute(
                """
                UPDATE patients
                SET BMI = ?
                WHERE BMI <= 0
                """,
                (round(median_bmi, 1),),
            )
            conn.commit()

    def get_all_ids(self) -> List[int]:
        with self._get_connection() as conn:
            rows = conn.execute(
                """
                SELECT patient_id
                FROM patients
                ORDER BY patient_id
                """
            ).fetchall()

        return [row[0] for row in rows]

    def get_patient(self, patient_id: int) -> Optional[Dict[str, float]]:
        with self._get_connection() as conn:
            row = conn.execute(
                """
                SELECT Glucose, BMI, Age, BloodPressure
                FROM patients
                WHERE patient_id = ?
                """,
                (patient_id,),
            ).fetchone()

        if row is None:
            return None

        return {
            "Glucose": float(row[0]),
            "BMI": float(row[1]),
            "Age": float(row[2]),
            "BloodPressure": float(row[3]),
        }

    def update_patient(
        self,
        patient_id: int,
        updated_metrics: Dict[str, float],
    ) -> bool:
        current_patient = self.get_patient(patient_id)

        if current_patient is None:
            return False

        current_patient.update(updated_metrics)

        with self._get_connection() as conn:
            cursor = conn.execute(
                """
                UPDATE patients
                SET
                    Glucose = ?,
                    BMI = ?,
                    Age = ?,
                    BloodPressure = ?
                WHERE patient_id = ?
                """,
                (
                    current_patient["Glucose"],
                    current_patient["BMI"],
                    current_patient["Age"],
                    current_patient["BloodPressure"],
                    patient_id,
                ),
            )
            conn.commit()

        return cursor.rowcount > 0


# =====================================================================
# 2. BUSINESS LOGIC LAYER
# =====================================================================
class ClinicalRiskService:
    """Handles clinical decision rules, point scoring, and risk categorization."""

    THRESHOLDS = {
        "Glucose": (100.0, 125.0),
        "BMI": (25.0, 29.9),
        "Age": (35.0, 55.0),
        "BloodPressure": (120.0, 130.0),
    }

    def calculate_metric_score(self, metric_name: str, value: float) -> int:
        if metric_name not in self.THRESHOLDS:
            return 0

        low_max, med_max = self.THRESHOLDS[metric_name]

        if value <= low_max:
            return 0
        elif value <= med_max:
            return 1
        return 2

    def evaluate_patient_risk(
        self,
        metrics: Dict[str, float],
    ) -> Tuple[int, str]:
        total_score = sum(
            self.calculate_metric_score(metric, value)
            for metric, value in metrics.items()
        )

        if total_score <= 2:
            category = "Low Risk"
        elif total_score <= 5:
            category = "Moderate Risk"
        else:
            category = "High Risk"

        return total_score, category


# =====================================================================
# 3. STREAMLIT PAGE CONFIG
# =====================================================================
st.set_page_config(
    page_title="Diabetes Risk Scoring System",
    page_icon="🩺",
    layout="centered",
)

st.title("🩺 DIABETES RISK SCORING SYSTEM")
st.caption("Clinical Risk Assessment")


# =====================================================================
# 4. SESSION STATE
# =====================================================================
if "model" not in st.session_state:
    st.session_state.model = PatientModel()

if "service" not in st.session_state:
    st.session_state.service = ClinicalRiskService()

if "selected_patient" not in st.session_state:
    st.session_state.selected_patient = (
        st.session_state.model.get_all_ids()[0]
    )

if "last_result" not in st.session_state:
    st.session_state.last_result = None


model: PatientModel = st.session_state.model
service: ClinicalRiskService = st.session_state.service


# =====================================================================
# 5. PATIENT SELECTION
# =====================================================================
st.subheader("Patient Selection")

patient_ids = model.get_all_ids()

selected_patient = st.selectbox(
    "Patient ID",
    options=patient_ids,
    index=patient_ids.index(st.session_state.selected_patient),
)

if selected_patient != st.session_state.selected_patient:
    st.session_state.selected_patient = selected_patient
    st.session_state.last_result = None

patient = model.get_patient(selected_patient)

if patient is None:
    st.error("Patient not found.")
    st.stop()


# =====================================================================
# 6. CLINICAL INPUTS
# =====================================================================
st.subheader("Clinical Measurements")

col1, col2 = st.columns(2)

with col1:
    glucose = st.number_input(
        "Fasting Blood Glucose (mg/dL)",
        min_value=0.0,
        value=float(patient["Glucose"]),
        step=1.0,
    )

    age = st.number_input(
        "Age (years)",
        min_value=0.0,
        value=float(patient["Age"]),
        step=1.0,
    )

with col2:
    bmi = st.number_input(
        "BMI (kg/m²)",
        min_value=0.0,
        value=float(patient["BMI"]),
        step=0.1,
    )

    blood_pressure = st.number_input(
        "Blood Pressure (mmHg)",
        min_value=0.0,
        value=float(patient["BloodPressure"]),
        step=1.0,
    )


metrics = {
    "Glucose": float(glucose),
    "BMI": float(bmi),
    "Age": float(age),
    "BloodPressure": float(blood_pressure),
}


# =====================================================================
# 7. ACTION BUTTONS
# =====================================================================
button_col1, button_col2 = st.columns(2)

with button_col1:
    calculate_clicked = st.button(
        "Calculate Risk",
        use_container_width=True,
        type="primary",
    )

with button_col2:
    save_clicked = st.button(
        "Save Measurements",
        use_container_width=True,
    )


if save_clicked:
    updated = model.update_patient(selected_patient, metrics)

    if updated:
        st.success(
            f"Measurements for Patient {selected_patient} "
            f"were saved to SQLite successfully."
        )
    else:
        st.error("Unable to save measurements.")


if calculate_clicked:
    model.update_patient(selected_patient, metrics)

    score, category = service.evaluate_patient_risk(metrics)

    st.session_state.last_result = {
        "patient_id": selected_patient,
        "score": score,
        "category": category,
    }


# =====================================================================
# 8. DIAGNOSTIC RISK REPORT
# =====================================================================
if st.session_state.last_result is not None:
    result = st.session_state.last_result

    st.divider()

    with st.container(border=True):
        st.markdown("### 🩺 DIAGNOSTIC RISK REPORT")
        st.caption("DIABETES RISK ASSESSMENT")

        st.markdown("---")

        col_report1, col_report2 = st.columns([1, 2])

        with col_report1:
            st.markdown("**Patient ID**")
            st.markdown("**Cumulative Score**")
            st.markdown("**Risk Category**")

        with col_report2:
            st.markdown(f"{result['patient_id']}")
            st.markdown(f"{result['score']} pts")
            st.markdown(f"**{result['category'].upper()}**")

        st.markdown("---")

        if result["category"] == "High Risk":
            st.error(
                f"⚠️ HIGH RISK — Score {result['score']} pts"
            )
        elif result["category"] == "Moderate Risk":
            st.warning(
                f"⚠️ MODERATE RISK — Score {result['score']} pts"
            )
        else:
            st.success(
                f"LOW RISK — Score {result['score']} pts"
            )

        st.markdown("---")
        st.markdown(
            "**🔒 CONFIDENTIAL MEDICAL INFORMATION**  \n"
            "**FOR AUTHORIZED USE ONLY**"
        )


# =====================================================================
# 9. DATABASE INFORMATION
# =====================================================================
st.divider()
st.caption("Persistent database: diabetes_patients.db (SQLite3)")
st.caption(
    "Saved measurements remain available after the application is closed."
)
