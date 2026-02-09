def process_interfaces(vendor_a, interface_a, vendor_b, interface_b, type_interface_a, type_interface_b, form):
    # 1. Primeiro rodamos a VALIDAÇÃO (Se houver erro, ele avisa o form)
    validate_interface_format(vendor_a, interface_a, type_interface_a, 'interface_a', form)
    validate_interface_format(vendor_b, interface_b, type_interface_b, 'interface_b', form)

    # 2. SÓ formatamos e colocamos no cleaned_data se NÃO houver erros
    if not form.errors:
        form.cleaned_data['interface_a'] = format_interface(vendor_a, type_interface_a, interface_a)
        form.cleaned_data['interface_b'] = format_interface(vendor_b, type_interface_b, interface_b)


        import re

def validate_interface_format(vendor, interface, type_interface, interface_name, form):
    # 1. Se estiver vazio, não faz nada
    if not vendor or not interface or not type_interface:
        return

    # 2. PADRONIZAÇÃO (O segredo para não barrar o que é certo)
    # Transformamos "GigabitEthernet" em "gi", "Nokia" em "nokia", etc.
    v = str(vendor).strip().lower()
    t_raw = str(type_interface).strip().lower()
    
    # Mapeia os nomes longos do formulário para as chaves do dicionário
    mapping = {
        'gigabitethernet': 'gi', 'tengige': 'te', 'hundredgige': 'hu',
        '100ge': 'hu', 'bundle-ether': 'lag', 'eth-trunk': 'lag',
        'lag': 'lag', 'pw': 'pw'
    }
    t = mapping.get(t_raw, t_raw) # Se não achar no mapa, usa o que veio

    # 3. LIMPEZA (Usa sua função para tirar prefixos se o usuário colou algo pronto)
    clean_iface = remove_prefix(str(interface).strip())

    # --- BARREIRA DE SEGURANÇA (Bloqueia datas e lixo) ---
    # Se tiver espaço, vírgula ou formato de data (2021-...), barra na hora
    if ',' in clean_iface or ' ' in clean_iface or re.match(r'^\d{4}-\d{2}', clean_iface):
        form.add_error(interface_name, "Formato inválido. Interfaces não podem ser datas ou conter vírgulas.")
        return

    # 4. PADRÕES DE REGEX (Melhorados para aceitar 2/1/21)
    pattern_cisco = r'^\d+/\d+/\d+(?:/\d+)?$' # Aceita 3 ou 4 níveis
    pattern_nokia = r'^(?:esat-)?\d+/\d+(?:/\d+|/c\d+)?$'
    pattern_huawei = r'^\d+/\d+/\d+$'
    pattern_lag_pw = r'^\d+$'

    vendor_patterns = {
        'nokia':  {'gi': pattern_nokia, 'te': pattern_nokia, 'hu': pattern_nokia, 'lag': pattern_lag_pw, 'pw': pattern_lag_pw},
        'huawei': {'gi': pattern_huawei, 'te': pattern_huawei, 'hu': pattern_huawei, 'lag': pattern_lag_pw, 'pw': pattern_lag_pw},
        'cisco':  {'gi': pattern_cisco, 'te': pattern_cisco, 'hu': pattern_cisco, 'lag': pattern_lag_pw, 'pw': pattern_lag_pw},
    }

    # 5. VALIDAÇÃO FINAL
    # Se o vendor existir no dicionário
    if v in vendor_patterns:
        pattern = vendor_patterns[v].get(t)
        if pattern:
            # Comparamos a interface LIMPA com o padrão
            if not re.fullmatch(pattern, clean_iface):
                msg = f"Formato inválido para {vendor}. Exemplo: "
                msg += "0/1/2" if v == 'huawei' else "1/1/1"
                form.add_error(interface_name, msg)
        else:
            # Caso o tipo não esteja no dicionário, tentamos uma validação genérica de barras
            if not re.fullmatch(r'^\d+(/\d+)+$', clean_iface) and t not in ['lag', 'pw']:
                form.add_error(interface_name, "Interface fora do padrão técnico.")
