import itertools
import string
import subprocess
import os
from concurrent.futures import ThreadPoolExecutor
import tkinter as tk
from tkinter import filedialog, messagebox
import threading

# Função para tentar abrir o arquivo RAR com uma senha
def tentar_abrir_arquivo_rar(arquivo_rar, senha, output_text):
    try:
        # Usando o WinRAR para tentar extrair o arquivo com a senha fornecida
        resultado = subprocess.run(
            ['rar', 'x', '-p' + senha, arquivo_rar, 'test'],  # x para extrair, -p para senha, 'test' para não extrair arquivos
            stdout=subprocess.PIPE,
            stderr=subprocess.PIPE,
            text=True
        )
        
        # Se não houver erro ao tentar extrair o arquivo, a senha é válida
        if "All OK" in resultado.stdout:
            print(f"Senha encontrada: {senha}")
            output_text.insert(tk.END, f"Senha encontrada: {senha}\n")
            output_text.yview(tk.END)  # Rolando para o final do Text widget
            return senha  # Retorna a senha correta
        else:
            return None
    except Exception as e:
        print(f"Erro ao tentar a senha {senha}: {e}")
        return None

# Função de força bruta para gerar senhas
def gerar_senhas(comprimento_maximo=4, output_text=None):
    caracteres = string.ascii_letters + string.digits
    for comprimento in range(1, comprimento_maximo + 1):
        for senha in itertools.product(caracteres, repeat=comprimento):
            senha_str = ''.join(senha)
            if output_text:
                output_text.insert(tk.END, f"Gerando senha: {senha_str}\n")
                output_text.yview(tk.END)  # Rolando para o final do Text widget
            yield senha_str

# Função para dividir o gerador de senhas em partes
def dividir_senhas(senhas, num_threads=16):
    # Divide o gerador de senhas em 'num_threads' partes
    lista_senhas = list(senhas)
    tamanho = len(lista_senhas)
    tamanho_por_thread = tamanho // num_threads
    for i in range(0, tamanho, tamanho_por_thread):
        yield lista_senhas[i:i + tamanho_por_thread]

# Função principal para tentar as senhas usando múltiplas threads
def brute_force_rar(arquivo_rar, comprimento_maximo, num_threads, output_text):
    senhas = gerar_senhas(comprimento_maximo, output_text)
    with ThreadPoolExecutor(max_workers=num_threads) as executor:
        for senha_em_tarefas in dividir_senhas(senhas, num_threads):
            # Para cada conjunto de senhas, distribuímos as tentativas para as threads
            futuros = [executor.submit(tentar_abrir_arquivo_rar, arquivo_rar, senha, output_text) for senha in senha_em_tarefas]
            for futuro in futuros:
                senha_encontrada = futuro.result()
                if senha_encontrada:
                    return senha_encontrada  # Retorna a senha encontrada

# Função para selecionar o arquivo RAR
def escolher_arquivo():
    arquivo = filedialog.askopenfilename(title="Escolha o arquivo RAR", filetypes=[("Arquivos RAR", "*.rar")])
    if arquivo:
        arquivo_entry.delete(0, tk.END)
        arquivo_entry.insert(0, arquivo)

# Função para a interface gráfica
def iniciar_forca_bruta():
    arquivo_rar = arquivo_entry.get()
    comprimento_maximo = int(comprimento_entry.get())
    num_threads = int(threads_entry.get())
    
    if not os.path.exists(arquivo_rar):
        messagebox.showerror("Erro", "O arquivo RAR especificado não foi encontrado.")
        return
    
    output_text.delete(1.0, tk.END)  # Limpa o campo de saída
    output_text.insert(tk.END, "Iniciando força bruta...\n")

    # Rodar o processo de força bruta em uma thread separada para não travar a interface
    def worker():
        senha_encontrada = brute_force_rar(arquivo_rar, comprimento_maximo, num_threads, output_text)

        if senha_encontrada:
            output_text.insert(tk.END, f"Senha encontrada: {senha_encontrada}\n")
        else:
            output_text.insert(tk.END, "Senha não encontrada.\n")

    threading.Thread(target=worker, daemon=True).start()

# Criando a interface gráfica
root = tk.Tk()
root.title("Brute Force RAR Password")

# Labels e entradas
arquivo_label = tk.Label(root, text="Caminho do arquivo RAR:")
arquivo_label.pack(pady=5)

# Campo de texto para caminho do arquivo
arquivo_entry = tk.Entry(root, width=50)
arquivo_entry.pack(pady=5)

# Botão para procurar arquivo
procurar_button = tk.Button(root, text="Procurar Arquivo", command=escolher_arquivo)
procurar_button.pack(pady=5)

comprimento_label = tk.Label(root, text="Comprimento máximo da senha:")
comprimento_label.pack(pady=5)
comprimento_entry = tk.Entry(root, width=10)
comprimento_entry.pack(pady=5)

threads_label = tk.Label(root, text="Número de threads (ex: 16):")
threads_label.pack(pady=5)
threads_entry = tk.Entry(root, width=10)
threads_entry.pack(pady=5)

# Botão para iniciar a força bruta
iniciar_button = tk.Button(root, text="Iniciar Força Bruta", command=iniciar_forca_bruta)
iniciar_button.pack(pady=10)

# Caixa de texto para mostrar as senhas geradas e o progresso
output_text = tk.Text(root, width=80, height=20)
output_text.pack(pady=5)

# Inicia o loop principal da interface gráfica
root.mainloop()
