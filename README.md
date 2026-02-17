import streamlit as st
import pandas as pd
import plotly.express as px
from fpdf import FPDF
from datetime import datetime
import os
from streamlit_gsheets import GSheetsConnection

# --- 1. CONFIGURACIÓN DEL SISTEMA ---
st.set_page_config(page_title="Gestión Clínica Retina", layout="wide", page_icon="👁️")

# CARPETA TEMPORAL PARA IMÁGENES (Para PDF)
CARPETA_IMG = "temp_imagenes_oct"
if not os.path.exists(CARPETA_IMG):
    os.makedirs(CARPETA_IMG)

# LISTA DE MOMENTOS
MOMENTOS = ["Basal"] + [f"Inyección #{i}" for i in range(1, 13)]

# CONVERSOR SNELLEN A LOGMAR
SNELLEN_MAP = {
    "20/20": 0.00, "20/25": 0.10, "20/30": 0.18, "20/40": 0.30,
    "20/50": 0.40, "20/60": 0.48, "20/70": 0.54, "20/80": 0.60,
    "20/100": 0.70, "20/125": 0.80, "20/150": 0.88, "20/200": 1.00,
    "20/400": 1.30, "Cuenta Dedos": 1.90, "Mov. Manos": 2.30, "PL": 3.00
}

# --- 2. GESTIÓN DE DATOS (NUBE) ---
def cargar_datos():
    try:
        conn = st.connection("gsheets", type=GSheetsConnection)
        return conn.read(ttl=0)
    except:
        return pd.DataFrame()

def guardar_datos_nube(nuevo_registro):
    try:
        conn = st.connection("gsheets", type=GSheetsConnection)
        df_actual = conn.read(ttl=0)
        df_nuevo = pd.DataFrame([nuevo_registro])
        df_final = pd.concat([df_actual, df_nuevo], ignore_index=True)
        conn.update(data=df_final)
        return True
    except Exception as e:
        st.error(f"Error de conexión: {e}")
        return False

# --- 3. FUNCIONES AUXILIARES ---
class PDFReport(FPDF):
    def header(self):
        self.set_font('Arial', 'B', 14)
        self.cell(0, 10, 'PROTOCOLO DE INVESTIGACION - RETINA', 0, 1, 'C')
        self.ln(5)
    def footer(self):
        self.set_y(-15)
        self.set_font('Arial', 'I', 8)
        self.cell(0, 10, f'Página {self.page_no()}', 0, 0, 'C')

def generar_pdf_completo(df_paciente, id_paciente):
    pdf = PDFReport()
    pdf.add_page()
    pdf.set_font("Arial", size=10)
    
    # Encabezado Paciente
    pdf.set_fill_color(220, 230, 241)
    pdf.set_font("Arial", 'B', 12)
    # Extraemos datos fijos del primer registro disponible
    reg_base = df_paciente.iloc[-1] 
    pdf.cell(0, 10, f"ID: {id_paciente} | Doc: {reg_base.get('Tipo_Doc','')} | Genero: {reg_base.get('Genero','')}", 0, 1, 'L', fill=True)
    pdf.set_font("Arial", size=9)
    pdf.multi_cell(0, 5, f"Entidad: {reg_base.get('Entidad_Salud','')} ({reg_base.get('Regimen','')}) | Diabetes: {reg_base.get('Anos_Diabetes','0')} años | HbA1c: {reg_base.get('HbA1c','%')}")
    pdf.ln(5)
    
    for i, row in df_paciente.iterrows():
        # Visita
        pdf.set_font("Arial", 'B', 11)
        pdf.set_fill_color(240, 240, 240)
        pdf.cell(0, 8, f"VISITA: {row.get('Momento')} | Fecha App: {row.get('Fecha_Aplicacion')}", 1, 1, 'L', fill=True)
        
        # Datos
        pdf.set_font("Arial", '', 9)
        txt = f"Dx: {row.get('Dx_Ppal')} ({row.get('Cod_Dx_Ppal')}) | Medico: {row.get('Medico')}\n"
        txt += f"Tx: {row.get('Medicamento')} ({row.get('Dosis')})\n"
        txt += f"AV OD: {row.get('AV_OD_LogMAR')} | AV OI: {row.get('AV_OI_LogMAR')} | Fecha AV: {row.get('Fecha_Toma_AV')}\n"
        txt += f"OCT OD: {row.get('Espesor_OD')}um (SQI:{row.get('SQI_OD')}) | OCT OI: {row.get('Espesor_OI')}um"
        pdf.multi_cell(0, 5, txt, 1)
        
        # Hallazgos
        hallazgos = []
        for h in ['Atrofia', 'Fibrosis', 'SRF', 'IRF', 'Edema', 'Quiste', 'Hemorragia']:
            if row.get(f'Biom_{h}') == 'Sí': hallazgos.append(h.upper())
            
        pdf.set_font("Arial", 'I', 9)
        pdf.cell(0, 6, f"HALLAZGOS: {', '.join(hallazgos) if hallazgos else 'Negativo'}", 1, 1)
        pdf.ln(3)

    return pdf.output(dest='S').encode('latin-1')

def analizar_protocolo(datos, av_previa):
    if datos['Biom_Atrofia'] or datos['Biom_Fibrosis']:
        return "SUSPENDER (Daño Estructural)", "error"
    if av_previa is not None:
        try:
            delta = float(datos['AV_OD_LogMAR']) - float(av_previa)
            if delta >= 0.2:
                return f"ALERTA (Pérdida {delta:.2f} LogMAR)", "warning"
        except: pass
    return "CONTINUAR TRATAMIENTO", "success"

def check_password():
    if "password_correct" not in st.session_state:
        st.session_state.password_correct = False
    if not st.session_state.password_correct:
        st.header("🔒 Acceso Seguro - Cohorte Retina")
        c1, c2 = st.columns(2)
        user = c1.text_input("Usuario")
        pwd = c2.text_input("Contraseña", type="password")
        if st.button("Entrar"):
            if user == "admin" and pwd == "admin123":
                st.session_state.password_correct = True
                st.session_state.role = "admin"
                st.rerun()
            elif user == "usuario" and pwd == "digitador1":
                st.session_state.password_correct = True
                st.session_state.role = "digitador"
                st.rerun()
            else: st.error("Error de credenciales")
        return False
    return True

# --- 4. INTERFAZ ---
if check_password():
    st.sidebar.title(f"Rol: {st.session_state.role.upper()}")
    opciones = ["📝 Registro Clínico"]
    if st.session_state.role == "admin": opciones += ["📊 Tablero & PDF", "🤖 IA"]
    menu = st.sidebar.radio("Menú", opciones)

    if menu == "📝 Registro Clínico":
        st.title("📝 Ficha de Captura - Retina")
        
        # Historial Rápido
        df_cloud = cargar_datos()
        c1, c2 = st.columns(2)
        id_paciente = c1.text_input("Número de Identificación (ID) *")
        momento = c2.selectbox("Momento del tratamiento", MOMENTOS)
        
        av_prev = None
        if id_paciente and not df_cloud.empty and 'ID' in df_cloud.columns:
            f = df_cloud[df_cloud['ID'] == id_paciente]
            if not f.empty:
                last = f.iloc[-1]
                if 'AV_OD_LogMAR' in last: 
                    av_prev = last['AV_OD_LogMAR']
                    st.info(f"📋 Última Visión: {av_prev} LogMAR")

        with st.form("form_master"):
            # --- PESTAÑAS ORGANIZADAS ---
            tabs = st.tabs(["👤 Filiación", "🏥 Antecedentes", "💊 Tx & Dx", "👁️ Visión", "🔬 OCT", "🧬 Biomarcadores"])
            
            # 1. FILIACIÓN
            with tabs[0]:
                col1, col2, col3 = st.columns(3)
                tipo_doc = col1.selectbox("Tipo Documento", ["CC", "TI", "CE", "Pasaporte"])
                genero = col2.radio("Género", ["F", "M"], horizontal=True)
                tipo_pac = col3.selectbox("Tipo Paciente", ["Naive", "Recurrente", "Switch"])
                
                col4, col5 = st.columns(2)
                entidad = col4.text_input("Entidad de Salud (EPS/Seguro)")
                regimen = col5.radio("Régimen", ["Contributivo", "Subsidiado", "Especial/Otro"], horizontal=True)
                
                st.divider()
                col6, col7 = st.columns(2)
                medico = col6.text_input("Médico Tratante")
                fecha_app = col7.date_input("Fecha de Aplicación")

            # 2. ANTECEDENTES Y METABÓLICO
            with tabs[1]:
                c_met1, c_met2 = st.columns(2)
                anos_dm = c_met1.number_input("Años con Diabetes", 0, 80)
                hba1c = c_met2.number_input("Hemoglobina Glicosilada (HbA1c %)", 0.0, 20.0, step=0.1)
                
                st.subheader("Antecedentes")
                ant_ocu = st.text_area("Ant. Oculares")
                ant_fam = st.text_area("Ant. Familiares")
                ant_qx = st.text_area("Ant. Quirúrgicos Oculares")

            # 3. DX Y TRATAMIENTO
            with tabs[2]:
                cd1, cd2 = st.columns(2)
                dx_ppal = cd1.text_input("Dx Principal (Descripción)")
                cod_dx_ppal = cd1.text_input("Cód. CIE-10 Ppal")
                
                dx_2 = cd2.text_input("Dx Secundario (Descripción)")
                cod_dx_2 = cd2.text_input("Cód. CIE-10 Secundario")
                
                st.divider()
                ct1, ct2 = st.columns(2)
                medicamento = ct1.selectbox("Medicamento a aplicar", ["Aflibercept", "Ranibizumab", "Bevacizumab", "Faricimab", "Brolucizumab"])
                dosis = ct2.text_input("Dosis", "0.05 ml")

            # 4. FUNCIÓN VISUAL
            with tabs[3]:
                cf1, cf2, cf3 = st.columns(3)
                fecha_av = cf1.date_input("Fecha de toma de AV")
                ojo_int = cf2.radio("Ojo Intervenido", ["OD", "OI"], horizontal=True)
                tipo_av = cf3.selectbox("Tipo", ["Lejos", "Cerca"])
                
                cf4, cf5 = st.columns(2)
                cartilla = cf4.selectbox("Cartilla", ["Snellen", "ETDRS"])
                distancia = cf5.text_input("Distancia", "6m")
                
                st.subheader("Registro de Agudeza Visual")
                col_od, col_oi = st.columns(2)
                with col_od:
                    st.markdown("**Ojo Derecho (OD)**")
                    s_od = st.selectbox("Snellen OD", list(SNELLEN_MAP.keys()), index=3)
                    log_od = SNELLEN_MAP[s_od]
                    st.caption(f"LogMAR: {log_od}")
                with col_oi:
                    st.markdown("**Ojo Izquierdo (OI)**")
                    s_oi = st.selectbox("Snellen OI", list(SNELLEN_MAP.keys()), index=3)
                    log_oi = SNELLEN_MAP[s_oi]
                    st.caption(f"LogMAR: {log_oi}")

            # 5. OCT
            with tabs[4]:
                ce1, ci1 = st.columns(2)
                equipo = ce1.selectbox("Equipo", ["Heidelberg", "Cirrus", "Topcon", "Optovue"])
                img_oct = ci1.file_uploader("Imagen OCT (Evidencia)")
                
                co1, co2 = st.columns(2)
                with co1:
                    esp_od = st.number_input("Espesor Central OD (um)", 0, 1000)
                    sqi_od = st.number_input("SQI OD", 0, 100)
                    ssi_od = st.number_input("SSI OD", 0, 100)
                with co2:
                    esp_oi = st.number_input("Espesor Central OI (um)", 0, 1000)
                    sqi_oi = st.number_input("SQI OI", 0, 100)
                    ssi_oi = st.number_input("SSI OI", 0, 100)

            # 6. BIOMARCADORES
            with tabs[5]:
                t_mnv = st.selectbox("Tipo MNV", ["1", "2", "3", "Polipoidal", "N/A"])
                
                st.error("⛔ Criterios Suspensión")
                cb1, cb2 = st.columns(2)
                b_atrofia = cb1.checkbox("Atrofia")
                b_fibrosis = cb2.checkbox("Fibrosis")
                
                st.warning("⚠️ Signos Actividad")
                cb3, cb4, cb5, cb6 = st.columns(4)
                b_srf = cb3.checkbox("SRF (Fluido Sub)")
                b_irf = cb4.checkbox("IRF (Fluido Intra)")
                b_edema = cb5.checkbox("Edema")
                b_quiste = cb6.checkbox("Quiste")
                
                st.info("ℹ️ Otros Hallazgos")
                cb7, cb8, cb9 = st.columns(3)
                b_hem = cb7.checkbox("Hemorragia")
                b_dep = cb8.checkbox("DEP")
                b_hrf = cb9.checkbox("HRFs")

            # --- GUARDADO ---
            btn = st.form_submit_button("💾 GUARDAR EN NUBE")
            
            if btn and id_paciente:
                # 1. Protocolo
                pack_prot = {"Biom_Atrofia": b_atrofia, "Biom_Fibrosis": b_fibrosis, "AV_OD_LogMAR": log_od}
                decision, tipo = analizar_protocolo(pack_prot, av_prev)
                
                if tipo == "error": st.error(decision)
                elif tipo == "warning": st.warning(decision)
                else: st.success(decision)
                
                # 2. Diccionario COMPLETO (Todas tus variables)
                nuevo = {
                    "ID": id_paciente, "Tipo_Doc": tipo_doc, "Genero": genero,
                    "Momento": momento, "Fecha_Aplicacion": str(fecha_app),
                    "Medico": medico, "Tipo_Paciente": tipo_pac,
                    "Entidad_Salud": entidad, "Regimen": regimen,
                    "Anos_Diabetes": anos_dm, "HbA1c": hba1c,
                    "Ant_Oculares": ant_ocu, "Ant_Familiares": ant_fam, "Ant_Qx": ant_qx,
                    "Medicamento": medicamento, "Dosis": dosis,
                    "Dx_Ppal": dx_ppal, "Cod_Dx_Ppal": cod_dx_ppal,
                    "Dx_Sec": dx_2, "Cod_Dx_Sec": cod_dx_2,
                    "Fecha_Toma_AV": str(fecha_av), "Ojo_Int": ojo_int, "Tipo_Vision": tipo_av,
                    "Cartilla": cartilla, "Distancia": distancia,
                    "AV_OD_LogMAR": log_od, "AV_OI_LogMAR": log_oi,
                    "Equipo_OCT": equipo, "Espesor_OD": esp_od, "Espesor_OI": esp_oi,
                    "SQI_OD": sqi_od, "SQI_OI": sqi_oi, "SSI_OD": ssi_od, "SSI_OI": ssi_oi,
                    "Tipo_MNV": t_mnv,
                    "Biom_Atrofia": "Sí" if b_atrofia else "No",
                    "Biom_Fibrosis": "Sí" if b_fibrosis else "No",
                    "Biom_SRF": "Sí" if b_srf else "No",
                    "Biom_IRF": "Sí" if b_irf else "No",
                    "Biom_Edema": "Sí" if b_edema else "No",
                    "Biom_Quiste": "Sí" if b_quiste else "No",
                    "Biom_Hemorragia": "Sí" if b_hem else "No",
                    "Biom_DEP": "Sí" if b_dep else "No",
                    "Biom_HRFs": "Sí" if b_hrf else "No",
                    "Decision_Sistema": decision
                }
                
                # 3. Enviar
                guardar_datos_nube(nuevo)
                st.toast("✅ Registro Exitoso")

    # MÓDULOS DE REPORTE (Mantienen estructura V7 con nuevas variables en PDF)
    elif menu == "📊 Tablero & PDF":
        st.title("📂 Reportes Cloud")
        df = cargar_datos()
        if not df.empty and 'ID' in df.columns:
            sel = st.selectbox("Paciente", df['ID'].unique())
            data_p = df[df['ID'] == sel]
            st.dataframe(data_p)
            
            if st.button("Generar PDF"):
                pdf_b = generar_pdf_completo(data_p, sel)
                st.download_button("📥 Descargar PDF", pdf_b, f"Reporte_{sel}.pdf", "application/pdf")
        else: st.warning("Sin datos.")

    elif menu == "🤖 IA":
        st.title("IA Analytics")
        df = cargar_datos()
        if not df.empty:
            x = st.selectbox("X", ["HbA1c", "Espesor_OD", "Anos_Diabetes"])
            y = st.selectbox("Y", ["AV_OD_LogMAR"])
            try:
                fig = px.scatter(df, x=x, y=y, trendline="ols")
                st.plotly_chart(fig)
            except: st.warning("Datos insuficientes")
