# app_streamlit.py
import streamlit as st
import csv
import random
from typing import List, Tuple, Dict, Optional
import math
import io

# =======================================================
# UTILS E LÓGICA DE GERAÇÃO (A Lógica original é mantida)
# =======================================================

# Função read_history adaptada para ler o arquivo enviado pelo Streamlit
@st.cache_data
def read_history(uploaded_file):
    """Lê e processa o arquivo CSV enviado, retornando a lista de sorteios."""
    if uploaded_file is None:
        return []

    # Decodifica o arquivo para texto
    string_data = uploaded_file.getvalue().decode("utf-8")
    
    # Usa o módulo csv para parsing
    reader = csv.reader(string_data.splitlines(), delimiter=',')
    rows = list(reader)
    hist = []
    
    start = 0
    if rows:
        first = rows[0]
        # Tentativa de identificar um cabeçalho
        if any(str(v).strip().lower() in {
            "n1","n2","n3","n4","n5","n6","n7","n8","n9","n10","n11","n12","n13","n14","n15",
            "sorteio","numero","num"
        } for v in first):
            start = 1

    for r in rows[start:]:
        nums = []
        # Tenta parsear cada coluna como número
        for v in r:
            try:
                x = int(str(v).strip())
                nums.append(x)
            except:
                continue
        # Verifica se tem pelo menos 15 números válidos
        if len(nums) >= 15 and all(1 <= n <= 25 for n in nums[:15]):
            hist.append(nums[:15])
            
    return hist

# --- (O RESTO DAS FUNÇÕES DE LÓGICA DO SEU CÓDIGO PYTHON ORIGINAL VÃO AQUI) ---
# NOTA: Para economizar espaço na resposta, estou omitindo as funções de lógica 
# (compute_frequency, build_hot_set, get_block, weighted_sample_without_replacement, etc.)
# Mas no arquivo final, elas devem ser COLADAS aqui, logo após o 'read_history'.

# Exemplo de adaptação:
def compute_frequency(history: List[List[int]]) -> Tuple[List[int], List[List[int]]]:
    # ... (Sua implementação original aqui)
    freq = [0] * 26
    pair_counts = [[0 for _ in range(26)] for __ in range(26)]
    for draw in history:
        for n in draw:
            if 1 <= n <= 25:
                freq[n] += 1
        for i in range(len(draw)):
            a = draw[i]
            for j in range(i+1, len(draw)):
                b = draw[j]
                x, y = (a, b) if a < b else (b, a)
                pair_counts[x][y] += 1
    return freq, pair_counts

def compute_sums_history(history: List[List[int]]) -> List[int]:
    return [sum(d) for d in history]
# ... (todas as outras funções...)
# [COLE A LÓGICA DE 'build_hot_set', 'get_block', 'percentile', 
# 'weighted_sample_without_replacement', 'random_block_quotas_fixed', 
# 'build_blocks', 'generate_candidate_with_hot', e 'generate_multiple' AQUI]
# ... (fim da lógica) ...


# =======================================================
# INTERFACE STREAMLIT
# =======================================================

def main_app():
    st.set_page_config(page_title="Lotofácil Gerador Avançado", layout="centered")
    st.title("🎰 Lotofácil Gerador Avançado")
    st.markdown("Cálculo de apostas otimizadas com base em seu histórico CSV.")
    
    # 1. Carregamento do Arquivo
    with st.sidebar:
        st.header("1. Carregar Histórico")
        uploaded_file = st.file_uploader(
            "Selecione o arquivo CSV de resultados:", 
            type=['csv']
        )
        
        # Lê o histórico e armazena em cache
        history_data = read_history(uploaded_file)
        
        if history_data:
            st.success(f"Histórico carregado: **{len(history_data)}** sorteios válidos.")
            
            # Pré-computar estatísticas
            freq, pair_counts = compute_frequency(history_data)
            sums_hist = compute_sums_history(history_data)
            
            # Armazenar estatísticas na sessão para evitar recalcular
            st.session_state['stats'] = {
                'history': history_data,
                'freq': freq,
                'pair_counts': pair_counts,
                'sums_hist': sums_hist,
            }
        else:
            st.warning("Aguardando o carregamento de um arquivo CSV válido.")
            st.session_state['stats'] = None
    
    # Verifica se os dados foram carregados antes de mostrar os controles
    if st.session_state['stats'] is None:
        st.info("Carregue seu arquivo CSV na barra lateral para começar.")
        return

    stats = st.session_state['stats']

    # 2. Parâmetros de Geração
    st.header("2. Parâmetros")
    
    col1, col2 = st.columns(2)
    
    with col1:
        generate_n = st.slider(
            "Quantidade de Apostas a Gerar:", 
            min_value=1, max_value=20, value=10
        )
        
    with col2:
        hot_ratio = st.slider(
            "Proporção de Números 'Quentes' (Hot Ratio):", 
            min_value=0.0, max_value=1.0, value=0.4, step=0.05
        )
        
    iterations = st.slider(
        "Iterações de Simulação (Qualidade):",
        min_value=500, max_value=10000, value=5000, step=500
    )
    
    # 3. Botão de Geração
    st.markdown("---")
    if st.button("🚀 GERAR PALPITES OTIMIZADOS", type="primary", use_container_width=True):
        
        # Recalcular hot set com base no ratio atual
        desired_hot = max(0, int(round(15 * hot_ratio)))
        hot_set_size = max(5, min(20, desired_hot * 3 if desired_hot > 0 else 7))
        hot_set = build_hot_set(stats['freq'], hot_set_size)
        
        with st.spinner('Processando milhares de simulações...'):
            results = generate_multiple(
                stats['freq'],
                hot_set,
                stats['pair_counts'],
                stats['sums_hist'],
                stats['history'],
                iterations,
                generate_n,
                generate_n, # top_k igual a generate_n
                hot_ratio
            )
        
        # 4. Exibir Resultados
        st.header("3. Apostas Sugeridas")
        st.caption(f"Baseado no Hot Set de {len(hot_set)} números mais frequentes.")

        for i, (score, palpite, mets) in enumerate(results):
            col_res, col_score = st.columns([4, 1])
            
            with col_res:
                # Criar a visualização dos números
                palpite_str = " ".join(f"**{n:02d}**" for n in palpite)
                st.markdown(f"**APOSTA {i + 1}:** {palpite_str}")
                
            with col_score:
                st.metric(label="Score Otimizado", value=f"{score:.2f}")

            # Exibir métricas
            st.expander(f"Detalhes da Aposta {i + 1}").write({
                "Ímpares": f"{mets.get('odd_count', 0)} (Paridade OK: {mets.get('parity_ok', False)})",
                "Soma Total": f"{mets.get('sum', 0)} (Na Faixa Ideal: {mets.get('sum_in_range', 0.0) == 1.0})",
                "Números Quentes (Hot)": mets.get('hot_count', 0),
                "Blocos Cobertos (Score)": mets.get('block_score', 0),
            })


if 'stats' not in st.session_state:
    st.session_state['stats'] = None

# A função main_app deve ser chamada apenas uma vez
main_app()
