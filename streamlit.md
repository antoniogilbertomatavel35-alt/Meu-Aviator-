import streamlit as st
import time
import hashlib
import hmac
import random
import datetime

# --- CONFIGURAÇÃO DA PÁGINA ---
st.set_page_config(page_title="Aviator Pro Online", page_icon="✈️", layout="wide")

# --- ESTILO VISUAL ---
st.markdown("""
    <style>
    .main { background-color: #0f1116; }
    .multiplicador-box { 
        font-size: 80px; font-weight: 800; color: #e91e63; 
        text-align: center; margin: 20px 0; font-family: 'Arial', sans-serif;
    }
    .historico-badge {
        display: inline-block; padding: 5px 12px; margin: 3px;
        border-radius: 20px; background: #262b37; color: #4caf50; font-weight: bold;
    }
    </style>
    """, unsafe_allow_html=True)

# --- INICIALIZAÇÃO ---
if 'banca' not in st.session_state: st.session_state.banca = 500.0
if 'historico_global' not in st.session_state: st.session_state.historico_global = [1.50, 2.10, 1.05]
if 'proximo_resultado' not in st.session_state: st.session_state.proximo_resultado = None

# --- FUNÇÃO DE CÁLCULO REAL ---
def gerar_resultado_automatico():
    hash_hex = hashlib.sha256(str(time.time()).encode()).hexdigest()
    hash_int = int(hash_hex[:13], 16)
    e = 2**52
    resultado = (100 * e - hash_int) / (e - hash_int)
    return max(1.0, round(resultado / 100, 2))

# --- SIDEBAR E PAINEL DO DONO ---
with st.sidebar:
    st.title("🛡️ Acesso Restrito")
    senha = st.text_input("Senha do Dono", type="password")
    
    if senha == "admin123": # <--- MUDE SUA SENHA AQUI
        st.success("Acesso Liberado")
        modo_manual = st.toggle("Ativar Manipulação")
        valor_forcado = st.number_input("Forçar Próximo Crash:", min_value=1.0, value=1.5)
        if modo_manual:
            st.session_state.proximo_resultado = valor_forcado
        else:
            st.session_state.proximo_resultado = None
    
    st.divider()
    st.metric("Sua Banca", f"R$ {st.session_state.banca:.2f}")

# --- INTERFACE PRINCIPAL ---
st.title("✈️ Aviator Pro")

# Histórico
cols = st.columns(10)
for i, val in enumerate(st.session_state.historico_global[-10:]):
    st.markdown(f'<span class="historico-badge">{val:.2f}x</span>', unsafe_allow_html=True)

col_game, col_bet = st.columns([2, 1])

with col_bet:
    st.subheader("Aposta")
    valor_aposta = st.number_input("Valor R$", min_value=1.0, value=10.0)
    auto_saque = st.number_input("Saque Automático (x)", min_value=1.1, value=2.0)
    if st.button("APOSTAR", use_container_width=True, type="primary"):
        if valor_aposta > st.session_state.banca:
            st.error("Saldo insuficiente")
        else:
            # Lógica da Rodada
            crash = st.session_state.proximo_resultado if st.session_state.proximo_resultado else gerar_resultado_automatico()
            
            placeholder_m = col_game.empty()
            mult = 1.00
            ganhou = False
            
            while mult <= crash:
                cor = "#e91e63" if mult < auto_saque else "#4caf50"
                placeholder_m.markdown(f'<p class="multiplicador-box" style="color: {cor};">{mult:.2f}x</p>', unsafe_allow_html=True)
                if mult >= auto_saque and not ganhou:
                    ganhou = True
                
                time.sleep(0.05)
                mult += 0.01 * (mult ** 1.1)
            
            if ganhou:
                st.session_state.banca += (valor_aposta * (auto_saque - 1))
                st.balloons()
            else:
                st.session_state.banca -= valor_aposta
            
            st.session_state.historico_global.append(crash)
            placeholder_m.markdown(f'<p class="multiplicador-box" style="color: #666;">💥 {crash:.2f}x</p>', unsafe_allow_html=True)
            time.sleep(2)
            st.rerun()

with col_game:
    if 'placeholder_m' not in locals():
        st.markdown(f'<p class="multiplicador-box" style="color: #333;">1.00x</p>', unsafe_allow_html=True)
