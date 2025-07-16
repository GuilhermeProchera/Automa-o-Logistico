# ==============================================================================
# =================== IMPORTAÇÃO DAS BIBLIOTECAS NECESSÁRIAS ===================
# ==============================================================================
import gspread
from google.oauth2.service_account import Credentials
from datetime import datetime
from flask import Flask, request
import traceback
import sendgrid
import requests # <-- Biblioteca para a API do Cargosnap
import base64   # <-- Biblioteca para o anexo de e-mail
from sendgrid.helpers.mail import (Mail, Attachment, FileContent, FileName, FileType, Disposition)

# ==============================================================================
# =================== INICIALIZAÇÃO DA APLICAÇÃO WEB ===========================
# ==============================================================================
app = Flask(__name__)

# ==============================================================================
# =================== ÁREA DE CONFIGURAÇÃO PRINCIPAL ===========================
# ==============================================================================
SERVICE_ACCOUNT_FILE = '/home/Prochera/automacao/credenciais.json'
SPREADSHEET_ID = '1NkHjKiscg-EcgBS5vT_Q-aBsPRybeqM2dcb0-HEKKSA'
# --- Configs de E-mail ---
SENDGRID_API_KEY = "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"
SENDER_EMAIL = "guilherme.prochera@elitecargas.com"
EMAIL_DESTINO = "guilherme.prochera@elitecargas.com"
# --- Configs API Cargosnap ---
CARGOSNAP_API_TOKEN = "ZWtjWTFXdW13Zzd0aE5FZVRRZlFVZDNLdTV0b0JpZ01fMzYwNw=="
CARGOSNAP_API_BASE_URL = "https://api.cargosnap.com/v1"
# --- MAPEAMENTO DE COLUNAS ---
NOME_DA_ABA = 'JULHO'
COLUNA_DA_REFERENCIA = 2; COLUNA_DO_CLIENTE = 3; COLUNA_CARROCERIA = 5
COLUNA_DA_UNIDADE = 6; COLUNA_PALETES = 7; COLUNA_TIPO_OPERACAO = 8
COLUNA_OBS = 13; COLUNA_DO_STATUS = 14

client, sheet = None, None

# ==============================================================================
# =================== FUNÇÕES DE AUTOMAÇÃO (CORE) ==============================
# ==============================================================================

def conectar_planilha():
    global client, sheet
    try:
        SCOPES = ['https://www.googleapis.com/auth/spreadsheets']
        creds = Credentials.from_service_account_file(SERVICE_ACCOUNT_FILE, scopes=SCOPES)
        client = gspread.authorize(creds)
        sheet = client.open_by_key(SPREADSHEET_ID).worksheet(NOME_DA_ABA)
        return True
    except Exception as e: return False

def encontrar_dados_carga(ref_code):
    try:
        cell = sheet.find(ref_code, in_column=COLUNA_DA_REFERENCIA)
        if cell:
            row_data = sheet.row_values(cell.row)
            def get_value(col_num): return row_data[col_num - 1].strip() if col_num and len(row_data) >= col_num else ""
            return {'linha': cell.row, 'referencia': get_value(COLUNA_DA_REFERENCIA), 'cliente': get_value(COLUNA_DO_CLIENTE), 'carroceria': get_value(COLUNA_CARROCERIA), 'unidade': get_value(COLUNA_DA_UNIDADE), 'paletes': get_value(COLUNA_PALETES), 'tipo_operacao': get_value(COLUNA_TIPO_OPERACAO), 'obs': get_value(COLUNA_OBS)}
    except gspread.CellNotFound: pass
    return None

def gerar_e_baixar_relatorio(ref_servico):
    """Usa a API do Cargosnap para baixar o arquivo do relatório (VERSÃO COM DEBUG)."""
    print(f"[DEBUG] Iniciando download do relatório para: {ref_servico}")
    headers = {'Authorization': f'Bearer {CARGOSNAP_API_TOKEN}'}
    endpoint_relatorio = f"{CARGOSNAP_API_BASE_URL}/jobs/{ref_servico}/reports"
    
    try:
        response = requests.get(endpoint_relatorio, headers=headers)
        
        # Vamos inspecionar a resposta da API em detalhes
        print(f"[DEBUG] Status da Resposta da API: {response.status_code}")
        print(f"[DEBUG] Headers da Resposta: {response.headers}")
        
        # Checagem de sucesso
        if response.status_code == 200:
            if response.content:
                print(f"[DEBUG] Resposta recebida com {len(response.content)} bytes.")
                content_type = response.headers.get('Content-Type', '')
                
                if 'application/pdf' in content_type:
                    filename = f"Relatorio_{ref_servico}.pdf"
                    print(f"[SUCESSO] Relatório PDF baixado e validado.")
                    return response.content, filename
                else:
                    # O que veio não é um PDF. Vamos ver o que é.
                    print(f"[ERRO CRÍTICO] O conteúdo recebido não é um PDF! Content-Type: {content_type}")
                    print(f"[DEBUG] Conteúdo recebido (texto): {response.text[:500]}") # Mostra os primeiros 500 caracteres
                    
            else:
                print("[ERRO CRÍTICO] A API retornou status 200, mas sem conteúdo (corpo vazio).")
        else:
            # A API retornou um erro (404, 401, 500, etc.)
            print(f"[ERRO CRÍTICO] A API do Cargosnap retornou um erro. Status: {response.status_code}")
            print(f"[DEBUG] Resposta de erro da API: {response.text}")

    except Exception as e:
        print(f"[ERRO CRÍTICO] Uma exceção ocorreu ao tentar chamar a API: {e}")
        traceback.print_exc() # Imprime o traceback completo do erro
        
    # Se chegou até aqui, algo falhou.
    return None, None

def enviar_email(assunto, corpo_html, file_content=None, filename=None):
    """Envia um e-mail de verdade, com ou sem anexo, usando SendGrid."""
    message = Mail(from_email=SENDER_EMAIL, to_emails=EMAIL_DESTINO, subject=assunto, html_content=corpo_html)
    if file_content and filename:
        encoded_file = base64.b64encode(file_content).decode()
        attachedFile = Attachment(FileContent(encoded_file), FileName(filename), FileType('application/pdf'), Disposition('attachment'))
        message.attachment = attachedFile
    try:
        sg = sendgrid.SendGridAPIClient(SENDGRID_API_KEY)
        response = sg.send(message)
        print(f"[INFO] E-mail enviado via SendGrid. Status: {response.status_code}")
    except Exception as e:
        print(f"[ERRO] Falha ao enviar e-mail via SendGrid: {e}")

def lidar_com_servico_criado(dados_planilha):
    print("[INFO] Executando fluxo de CRIAÇÃO de serviço.")
    assunto = f"Novo Serviço Criado: {dados_planilha['referencia']} - {dados_planilha['cliente']}"
    corpo_html = f"<p>Um novo fluxo de trabalho foi criado no Cargosnap.</p><ul><li><strong>Referência:</strong> {dados_planilha['referencia']}</li><li><strong>Tipo de Operação:</strong> {dados_planilha['tipo_operacao']}</li><li><strong>Cliente:</strong> {dados_planilha['cliente']}</li></ul><p>-- Notificação automática --</p>"
    enviar_email(assunto, corpo_html)
    print("[SUCESSO] Notificação de criação enviada.")

def lidar_com_servico_finalizado(dados_planilha):
    print("[INFO] Executando fluxo de FINALIZAÇÃO de serviço.")
    
    # 1. Baixa o relatório em PDF
    conteudo_relatorio, nome_arquivo = gerar_e_baixar_relatorio(dados_planilha['referencia'])

    # 2. Monta o corpo do e-mail
    veiculo_info = f"Container/Unidade: {dados_planilha['unidade']}" if dados_planilha['unidade'] else f"Carroceria: {dados_planilha['carroceria']}"
    assunto = f"{dados_planilha['tipo_operacao']} FINALIZADO: {dados_planilha['referencia']} - {dados_planilha['cliente']}"
    corpo_html = f"""
    <p>A operação foi concluída. Seguem os detalhes:</p>
    <pre style="font-family: monospace;">
{dados_planilha['tipo_operacao'].upper()}

Referência: {dados_planilha['referencia']}
{veiculo_info}
Cliente:    {dados_planilha['cliente']}
Paletes:    {dados_planilha['paletes']}
Obs:        {dados_planilha['obs']}
    </pre>
    <p>O relatório da operação encontra-se em anexo.</p>
    <p><em>-- E-mail enviado automaticamente --</em></p>
    """
    
    # 3. Envia o e-mail com o anexo
    enviar_email(assunto, corpo_html, conteudo_relatorio, nome_arquivo)

    # 4. Atualiza a planilha
    try:
        sheet.update_cell(dados_planilha['linha'], COLUNA_DO_STATUS, 'Concluído')
        print(f"[INFO] Linha {dados_planilha['linha']} marcada como 'Concluído'.")
    except Exception as e: print(f"[ERRO] Falha ao atualizar a planilha: {e}")
    
    print("[SUCESSO] Automação de finalização concluída.")

# ==============================================================================
# =================== PONTO DE ENTRADA DO WEBHOOK (ROTEADOR) ===================
# ==============================================================================
@app.route('/webhook', methods=['POST'])
def webhook_receiver():
    print(">>> Webhook recebido! <<<")
    dados_webhook = request.get_json(silent=True)
    if not dados_webhook: return "Erro: JSON Inválido.", 400

    ref_servico = dados_webhook.get('reference') or dados_webhook.get('file', {}).get('scan_code')
    if not ref_servico: return "Erro: campo 'reference' ausente.", 400
    
    if not conectar_planilha(): return "Erro interno.", 500

    dados_planilha = encontrar_dados_carga(ref_servico)
    if not dados_planilha: return "Referência não encontrada.", 404
    
    if dados_webhook.get('completed_at'):
        lidar_com_servico_finalizado(dados_planilha)
    else:
        lidar_com_servico_criado(dados_planilha)
    
    return "Webhook processado com sucesso.", 200
