import streamlit as st
import pandas as pd
import sqlite3
from datetime import datetime
import uuid

# --- CONFIGURACIÓN DE PÁGINA ---
st.set_page_config(page_title="VENEICA - IA Migratorio Mentor", layout="centered")

# --- ESTILOS PERSONALIZADOS ---
st.markdown("""
    <style>
    .stButton>button { width: 100%; border-radius: 20px; height: 3em; background-color: #1A365D; color: white; }
    .stAlert { border-radius: 15px; }
    .main { background-color: #f5f7f9; }
    </style>
    """, unsafe_allow_html=True)

# --- BASE DE DATOS (KPIs ANÓNIMOS) ---
def init_db():
    conn = sqlite3.connect('veneica_kpi.db')
    c = conn.cursor()
    c.execute('''CREATE TABLE IF NOT EXISTS logs 
                 (id TEXT, timestamp TEXT, tramite TEXT, hito_alcanzado TEXT, distrito TEXT)''')
    conn.commit()
    conn.close()

def log_kpi(tramite, hito, distrito):
    conn = sqlite3.connect('veneica_kpi.db')
    c = conn.cursor()
    c.execute("INSERT INTO logs VALUES (?,?,?,?,?)", 
              (str(uuid.uuid4()), datetime.now().strftime("%Y-%m-%d %H:%M:%S"), tramite, hito, distrito))
    conn.commit()
    conn.close()

init_db()

# --- LÓGICA DE TRÁMITES ---
TRAMITES_DATA = {
    "Citas": {
        "resumen": "Reserva de espacio presencial para trámites específicos.",
        "pasos": ["Ingresa a la Agencia Virtual.", "Selecciona 'Citas en línea'.", "Elige sede y horario disponible."],
        "checklist": ["Documento de identidad", "Número de recibo de pago (si aplica)", "Correo activo"],
        "errores": "No confirmar el correo electrónico tras reservar."
    },
    "Datos Biométricos": {
        "resumen": "Toma de huellas y foto tras iniciar un trámite de residencia.",
        "pasos": ["Verifica tu buzón electrónico.", "Imprime la cita de biométricos.", "Asiste 15 min antes."],
        "checklist": ["Cita impresa", "Pasaporte o cédula", "Recibo de pago"],
        "errores": "Llevar ropa de color claro (fondo suele ser blanco)."
    },
    "Renovación de CE": {
        "resumen": "Trámite anual para mantener la vigencia del carné.",
        "pasos": ["Pagar tasa en el Banco de la Nación.", "Validar requisitos en la Agencia Virtual.", "Subir documentos PDF."],
        "checklist": ["Recibo de pago Tasa 1435", "Declaración Jurada", "Pasaporte"],
        "errores": "Intentar renovar más de 30 días antes del vencimiento."
    }
}

# --- INTERFAZ ---
def main():
    if 'step' not in st.session_state:
        st.session_state.step = 'home'

    # PANTALLA: HOME
    if st.session_state.step == 'home':
        st.image("https://img.icons8.com/fluency/96/passport.png") # Placeholder para logo
        st.title("IA Migratorio Mentor – VENEICA")
        st.subheader("Tu guía paso a paso para trámites en Perú")
        st.info("⚠️ Esta es una herramienta orientativa. No guarda datos sensibles.")
        
        if st.button("Iniciar mi ruta"):
            st.session_state.step = 'privacy'

    # PANTALLA: PRIVACIDAD
    elif st.session_state.step == 'privacy':
        st.title("Privacidad y Consentimiento")
        st.write("""
        Para ayudarte, registraremos datos anónimos (trámite y ciudad). 
        **No pediremos números de documentos ni fotos.**
        """)
        distrito = st.selectbox("¿En qué ciudad/distrito te encuentras?", ["Lima", "Callao", "Arequipa", "Trujillo", "Otro"])
        
        if st.button("Acepto y Continuar"):
            st.session_state.distrito = distrito
            st.session_state.step = 'triage'

    # PANTALLA: TRIAGE
    elif st.session_state.step == 'triage':
        st.title("¿Cómo podemos ayudarte?")
        opcion = st.selectbox("Selecciona tu trámite:", list(TRAMITES_DATA.keys()) + ["Otro / No sé"])
        
        if st.button("Generar mi Ruta"):
            if opcion == "Otro / No sé":
                st.session_state.step = 'handoff'
            else:
                st.session_state.current_tramite = opcion
                log_kpi(opcion, "Triage Completado", st.session_state.distrito)
                st.session_state.step = 'ruta'

    # PANTALLA: RESULTADO DE RUTA
    elif st.session_state.step == 'ruta':
        data = TRAMITES_DATA[st.session_state.current_tramite]
        st.success(f"Ruta para: {st.session_state.current_tramite}")
        
        st.markdown(f"**Resumen:** {data['resumen']}")
        
        st.markdown("### 📋 Pasos a seguir")
        for i, paso in enumerate(data['pasos'], 1):
            st.write(f"{i}. {paso}")
            
        with st.expander("✅ Checklist de Requisitos (Genérico)"):
            for item in data['checklist']:
                st.checkbox(item)
            st.caption("Nota: Verificar requisitos oficiales en la Agencia Virtual.")

        st.warning(f"⚠️ **Evita este error:** {data['errores']}")
        
        col1, col2 = st.columns(2)
        with col1:
            if st.button("¡Logré este hito!"):
                log_kpi(st.session_state.current_tramite, "Hito Alcanzado", st.session_state.distrito)
                st.balloons()
        with col2:
            if st.button("Tengo una duda/error"):
                st.session_state.step = 'handoff'

        if st.button("Volver al inicio"):
            st.session_state.step = 'home'

    # PANTALLA: HANDOFF
    elif st.session_state.step == 'handoff':
        st.title("Hablar con un Agente")
        st.write("Tu caso requiere atención humana. Hemos generado un resumen para agilizar tu consulta.")
        
        codigo_resumen = f"VNE-{st.session_state.get('current_tramite', 'GENERAL')[:3].upper()}-{st.session_state.distrito[:3].upper()}"
        st.code(codigo_resumen, language="text")
        st.caption("Copia este código y pégalo en el chat.")
        
        st.markdown(f"[📲 WhatsApp Agente 1 (904 738 908)](https://wa.me/51904738908?text=Hola,%20mi%20codigo%20es%20{codigo_resumen})")
        st.markdown(f"[📲 WhatsApp Agente 2 (912 312 302)](https://wa.me/51912312302?text=Hola,%20mi%20codigo%20es%20{codigo_resumen})")
        
        st.markdown("---")
        st.markdown(f"[📝 Formulario Registro 2026](https://docs.google.com/forms/d/e/1FAIpQLSdO2t3t9aFkV92bSKEaVAXGxwXebIK_HE1w7uhtImJsKSWI_A/viewform?usp=dialog)")
        
        if st.button("Volver"):
            st.session_state.step = 'home'

if __name__ == "__main__":
    main()
