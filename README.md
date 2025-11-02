# ---------------------------
# 2) Butonlu Problem Çözme Aracı
# ---------------------------
import sympy as sp
import math
from ipywidgets import Textarea, Text, Button, Dropdown, HBox, VBox, Output, Checkbox, RadioButtons
from IPython.display import display, Markdown, HTML

# Yardımcı: degree-mode için trig dönüşümleri
def preprocess_degree_mode(s_expr: str, degree_mode: bool):
    """
    Eğer derece modundaysa, sin(x)->sin(rad(x)) gibi dönüşüm yap.
    Inverse trigler (asin, acos, atan) için burada dönüşüm yapmıyoruz; 
    sonuçları hesaplandıktan sonra dereceye çeviriyoruz.
    """
    if not degree_mode:
        return s_expr
    # Basit string tabanlı dönüşümler
    # Örn: sin( -> sin(rad(
    patterns = ['sin', 'cos', 'tan', 'sinh', 'cosh', 'tanh']
    out = s_expr
    for p in patterns:
        out = out.replace(f"{p}(", f"{p}(rad(")
    return out

# Simge için rad fonksiyonu: rad(x) = x*pi/180
rad = lambda x: x * sp.pi / 180

# Arayüz öğeleri
expr_input = Textarea(value="sin(30) + 2**3", placeholder="Örn: sin(30) veya 2*x + 3 = 7 veya integrate(x**2, x)", description="İfade:", layout={'width':'100%', 'height':'70px'})
mode_radio = RadioButtons(options=['Radyan','Derece'], value='Radyan', description='Mod:')
calc_btn = Button(description="Hesapla (Numerik)", button_style='info')
solve_btn = Button(description="Çöz (Solve)", button_style='success')
deriv_btn = Button(description="Türev (d/dx)", button_style='')
int_btn = Button(description="İntegral", button_style='')
simplify_btn = Button(description="Sadeleştir", button_style='')
steps_chk = Checkbox(value=False, description='Aşamaları Göster')
clear_btn = Button(description="Temizle", button_style='warning')

output = Output(layout={'border': '1px solid black'})

# İşlev: sympify güvenli isimler
safe_locals = {
    # trig ve matematik fonksiyonları
    'sin': sp.sin, 'cos': sp.cos, 'tan': sp.tan,
    'asin': sp.asin, 'acos': sp.acos, 'atan': sp.atan,
    'sinh': sp.sinh, 'cosh': sp.cosh, 'tanh': sp.tanh,
    'exp': sp.exp, 'log': sp.log, 'sqrt': sp.sqrt,
    'pi': sp.pi, 'E': sp.E, 'e': sp.E, 'tau': 2*sp.pi,
    'factorial': sp.factorial, 'fact': sp.factorial,
    'nCr': lambda n,r: sp.binomial(n,r),
    # destek için rad
    'rad': rad
}
# SymPy'nin tanımadığı 'j' yerine I
safe_locals['I'] = sp.I

# Yardımcı: ifade parse et
def parse_expr(s: str, degree_mode: bool):
    s_proc = preprocess_degree_mode(s, degree_mode)
    # SymPy'ye equation verildiğinde sol=sağ biçimini handle edeceğiz
    return s_proc

# Buton fonksiyonları
def show_output(msg, kind='text'):
    with output:
        if kind == 'text':
            print(msg)
        elif kind == 'md':
            display(Markdown(msg))
        else:
            print(msg)

def numeric_evaluate(btn=None):
    output.clear_output()
    s = expr_input.value.strip()
    deg = (mode_radio.value == 'Derece')
    if not s:
        show_output("Lütfen bir ifade gir.", 'text')
        return
    try:
        s_proc = parse_expr(s, deg)
        # Eğer ifade '=' içeriyorsa, numeric evaluate yerine solve öner
        if '=' in s and not any(k in s for k in ['solve','integrate','diff','limit']):
            show_output("Denklem görüldü. 'Çöz (Solve)' butonunu kullanmayı deneyin.", 'text')
            return
        # sympify ve sayısal değer (N)
        expr_sym = sp.sympify(s_proc, locals=safe_locals)
        # Eğer sonuç sembolik ise sayısallaştır
        val = sp.N(expr_sym)
        # Derece modu ve ters trigler: eğer kullanıcı asin/acos/atan kullandıysa ve derece modundaysa dereceye çevir
        if deg and any(tok in s for tok in ['asin(', 'acos(', 'atan(']):
            try:
                # val radyan cinsindense dereceye çevir
                val = sp.N(val * 180 / sp.pi)
            except Exception:
                pass
        show_output(f"Sonuç: {sp.pretty(val)}", 'text')
        if steps_chk.value:
            # Gösterilebilecek adımlar: sadeleştirme veya detay
            try:
                show_output("Sadeleştirilmiş: " + str(sp.simplify(expr_sym)), 'text')
            except Exception:
                pass
    except Exception as e:
        show_output(f"Hata: {e}", 'text')

def solve_action(btn=None):
    output.clear_output()
    s = expr_input.value.strip()
    deg = (mode_radio.value == 'Derece')
    if not s:
        show_output("Lütfen bir denklem veya ifade girin. Örn: 2*x+3=7", 'text')
        return
    try:
        # Denklem mi kontrol et
        if '=' in s:
            left, right = s.split('=',1)
            left_proc = parse_expr(left, deg)
            right_proc = parse_expr(right, deg)
            left_sym = sp.sympify(left_proc, locals=safe_locals)
            right_sym = sp.sympify(right_proc, locals=safe_locals)
            eq = sp.Eq(left_sym, right_sym)
            # Hangi değişkeni çözeceğimizi otomatik seç (ilk sembol)
            syms = list(eq.free_symbols)
            if not syms:
                show_output("Denklemde değişken bulunamadı.", 'text')
                return
            sol = sp.solve(eq, syms)
            show_output("Çözümler:", 'text')
            show_output(sol, 'text')
            if steps_chk.value:
                # possible to show steps using solve_univariate_... but SymPy adım adım sınırlı;
                show_output("Not: SymPy tüm problemlerde adım adım çözüm sunmayabilir.", 'text')
        else:
            # Eğer eşitlik yoksa solve(expression, symbol) deneyeceğiz
            expr_proc = parse_expr(s, deg)
            expr_sym = sp.sympify(expr_proc, locals=safe_locals)
            syms = list(expr_sym.free_symbols)
            if not syms:
                show_output("Denklem yok — ifade sembolik, doğrudan değerlendirin (Hesapla).", 'text')
                return
            # Denkleme dönüştür: expr_sym = 0
            sol = sp.solve(sp.Eq(expr_sym, 0), syms)
            show_output("Çözümler (expr = 0):", 'text')
            show_output(sol, 'text')
    except Exception as e:
        show_output(f"Hata: {e}", 'text')

def derivative_action(btn=None):
    output.clear_output()
    s = expr_input.value.strip()
    deg = (mode_radio.value == 'Derece')
    if not s:
        show_output("Lütfen türevini almak istediğiniz ifadeyi yazın, örn: x**2 veya sin(x).", 'text')
        return
    try:
        s_proc = parse_expr(s, deg)
        expr_sym = sp.sympify(s_proc, locals=safe_locals)
        syms = list(expr_sym.free_symbols)
        if not syms:
            show_output("Değişken bulunamadı. Örn: x kullanın.", 'text')
            return
        var = syms[0]
        d = sp.diff(expr_sym, var)
        show_output(f"d/d{var} {sp.pretty(expr_sym)} = {sp.pretty(d)}", 'text')
        if steps_chk.value:
            show_output("Sadeleştirilmiş türev: " + str(sp.simplify(d)), 'text')
    except Exception as e:
        show_output(f"Hata: {e}", 'text')

def integral_action(btn=None):
    output.clear_output()
    s = expr_input.value.strip()
    deg = (mode_radio.value == 'Derece')
    if not s:
        show_output("Lütfen integrali alınacak ifadeyi girin. Tek değişkenli integral için örn: x**2", 'text')
        return
    try:
        s_proc = parse_expr(s, deg)
        expr_sym = sp.sympify(s_proc, locals=safe_locals)
        syms = list(expr_sym.free_symbols)
        if not syms:
            # sabit için integral w.r.t x
            var = sp.Symbol('x')
        else:
            var = syms[0]
        integ = sp.integrate(expr_sym, var)
        show_output(f"∫ {sp.pretty(expr_sym)} d{var} = {sp.pretty(integ)} + C", 'text')
        if steps_chk.value:
            show_output("Sadeleştirilmiş integral: " + str(sp.simplify(integ)), 'text')
    except Exception as e:
        show_output(f"Hata: {e}", 'text')

def simplify_action(btn=None):
    output.clear_output()
    s = expr_input.value.strip()
    deg = (mode_radio.value == 'Derece')
    if not s:
        show_output("Lütfen sadeleştirilecek ifadeyi girin.", 'text')
        return
    try:
        s_proc = parse_expr(s, deg)
        expr_sym = sp.sympify(s_proc, locals=safe_locals)
        simp = sp.simplify(expr_sym)
        show_output(f"Sadeleştirilmiş: {sp.pretty(simp)}", 'text')
        if steps_chk.value:
            show_output("Expand: " + str(sp.expand(expr_sym)), 'text')
            show_output("Factor: " + str(sp.factor(expr_sym)), 'text')
    except Exception as e:
        show_output(f"Hata: {e}", 'text')

def clear_action(btn=None):
    output.clear_output()
    expr_input.value = ""
    show_output("Temizlendi.", 'text')

# Bağla
calc_btn.on_click(numeric_evaluate)
solve_btn.on_click(solve_action)
deriv_btn.on_click(derivative_action)
int_btn.on_click(integral_action)
simplify_btn.on_click(simplify_action)
clear_btn.on_click(clear_action)

# Düzenle ve göster
top_row = HBox([mode_radio, steps_chk, clear_btn])
mid_row = HBox([calc_btn, solve_btn, deriv_btn, int_btn, simplify_btn])
ui = VBox([Markdown("## Süper Gelişmiş Problem Çözme Aracı (Colab)"),
           Markdown("**Kılavuz:** Denklemler için `=` kullanın (örn `2*x+3=7`). Türev/integral için `x` değişkenini kullanın."),
           expr_input, top_row, mid_row, output])
display(ui)

# Başlangıç mesajı
with output:
    print("Arayüz hazır. İfadeyi yazıp ilgili butona basabilirsin.")


