# ultra_gelismis_mesai_hesaplayici.py (v5.3 - Global Fonksiyonlar - Hata Düzeltilmiş)
import csv
import json
from typing import List, Dict, Optional
import flet as ft
import pandas as pd
import os
from datetime import datetime

# --- Sabitler ---
KURAL_DOSYASI = "mesai_kurallari.json"

# --- Renk Sabitleri ---
RENK_PRİMER_MAVI = "#1976D2"
RENK_YESIL_INDIR = "#4CAF50"
RENK_GRI_ARKA_PLAN = "#FAFAFA"
RENK_KIRMIZI_UYARI = "#D32F2F"
RENK_BEYAZ = "#FFFFFF"
RENK_KIRMIZI_SIL = "#FF5252"
RENK_DUZENLE = "#FFC107"

# --- Kurallar Sınıfı ve Temel Fonksiyonlar ---
class KuralSeti:
    def __init__(self, ad: str, normal_aylik_saat: float, fazla_mesai_kat: float, bayram_kat: float):
        self.ad = ad
        self.NORMAL_AYLIK_SAAT = normal_aylik_saat
        self.FAZLA_MESAI_KAT = fazla_mesai_kat
        self.BAYRAM_KAT = bayram_kat
    
    def to_dict(self) -> Dict:
        return {
            "ad": self.ad, "normal_aylik_saat": self.NORMAL_AYLIK_SAAT,
            "fazla_mesai_kat": self.FAZLA_MESAI_KAT, "bayram_kat": self.BAYRAM_KAT
        }

def kurallari_yukle() -> Dict[str, KuralSeti]:
    try:
        with open(KURAL_DOSYASI, 'r', encoding='utf-8') as f:
            data = json.load(f)
            return {kural_ad: KuralSeti(**kural_data) for kural_ad, kural_data in data.items()}
    except (FileNotFoundError, json.JSONDecodeError):
        varsayilan_kurallar = {
            "Beyaz Yaka (Ofis)": KuralSeti("Beyaz Yaka (Ofis)", 160.0, 1.5, 2.0),
            "Mavi Yaka (Üretim)": KuralSeti("Mavi Yaka (Üretim)", 176.0, 1.6, 2.5)
        }
        kurallari_kaydet(varsayilan_kurallar)
        return varsayilan_kurallar

def kurallari_kaydet(kural_sozluk: Dict[str, KuralSeti]) -> None:
    kayit_data = {ad: kural.to_dict() for ad, kural in kural_sozluk.items()}
    with open(KURAL_DOSYASI, 'w', encoding='utf-8') as f:
        json.dump(kayit_data, f, indent=4, ensure_ascii=False)

def hesapla_ve_raporla(veri_listesi: List[Dict], kural_sozluk: Dict[str, KuralSeti]) -> pd.DataFrame:
    if not veri_listesi: return pd.DataFrame()
    df = pd.DataFrame(veri_listesi)
    
    df['Saatlik_Ucret'] = pd.to_numeric(df['Saatlik_Ucret'], errors='coerce').fillna(0).round(2)
    df['Toplam_Calisma_Saat'] = pd.to_numeric(df['Toplam_Calisma_Saat'], errors='coerce').fillna(0)
    df['Bayram_Calisma_Saat'] = pd.to_numeric(df['Bayram_Calisma_Saat'], errors='coerce').fillna(0)

    def mesai_saatleri(row):
        kural = kural_sozluk.get(row['Kural_Seti'])
        if kural:
            fm_saat = max(0, row['Toplam_Calisma_Saat'] - kural.NORMAL_AYLIK_SAAT)
            normal_saat = min(row['Toplam_Calisma_Saat'], kural.NORMAL_AYLIK_SAAT)
            return fm_saat, normal_saat
        return 0, 0

    fm_normal_saatler = df.apply(lambda row: mesai_saatleri(row), axis=1, result_type='expand')
    df['Fazla_Mesai_Saat'] = fm_normal_saatler[0]
    df['Normal_Mesai_Saat'] = fm_normal_saatler[1]
    
    def ucret_hesapla(row, katsayi_tipi):
        kural = kural_sozluk.get(row['Kural_Seti'])
        if not kural: return 0
        
        if katsayi_tipi == 'normal':
            return row['Normal_Mesai_Saat'] * row['Saatlik_Ucret']
        elif katsayi_tipi == 'fm':
            return row['Fazla_Mesai_Saat'] * row['Saatlik_Ucret'] * kural.FAZLA_MESAI_KAT
        elif katsayi_tipi == 'bayram':
            return row['Bayram_Calisma_Saat'] * row['Saatlik_Ucret'] * kural.BAYRAM_KAT
        return 0

    df['Normal_Mesai_Ucret'] = df.apply(lambda row: ucret_hesapla(row, 'normal'), axis=1).round(2)
    df['Fazla_Mesai_Ucret'] = df.apply(lambda row: ucret_hesapla(row, 'fm'), axis=1).round(2)
    df['Bayram_Mesai_Ucret'] = df.apply(lambda row: ucret_hesapla(row, 'bayram'), axis=1).round(2)
    
    df['Toplam_Ek_Odeme'] = df['Fazla_Mesai_Ucret'] + df['Bayram_Mesai_Ucret']
    df['Brüt_Maas_Tahmini'] = df['Normal_Mesai_Ucret'] + df['Toplam_Ek_Odeme']
    
    df['Toplam_Ek_Odeme'] = df['Toplam_Ek_Odeme'].round(2)
    df['Brüt_Maas_Tahmini'] = df['Brüt_Maas_Tahmini'].round(2)
    
    rapor_sutunlari = [
        'Ad_Soyad', 'Kural_Seti', 'Normal_Mesai_Saat', 'Fazla_Mesai_Saat', 'Bayram_Calisma_Saat',
        'Normal_Mesai_Ucret', 'Fazla_Mesai_Ucret', 'Bayram_Mesai_Ucret',
        'Toplam_Ek_Odeme', 'Brüt_Maas_Tahmini'
    ]
    
    return df[rapor_sutunlari]

def raporu_excel_kaydet(df: pd.DataFrame, dosya_adi: str) -> None:
    try:
        df.to_excel(dosya_adi, index=False, sheet_name='Puantaj Raporu')
    except Exception as e:
        raise Exception(f"Excel kaydetme hatası: {e}")

# ----------------------------------------------------------------------
# 2. GLOBAL DURUM VE OLAY İŞLEYİCİ FONKSİYONLAR
# ----------------------------------------------------------------------

# Uygulamanın durumunu tutan global sözlük
STATE = {
    'duzenlenen_kayit_index': None, 
    'personel_giris_listesi': [],
    'mevcut_kurallar': kurallari_yukle() 
}

def reset_form_action(form_elements: Dict, page: ft.Page):
    """Formu temizler ve Ekle moduna döndürür."""
    STATE['duzenlenen_kayit_index'] = None
    form_elements['txt_ad_soyad'].value = ""
    form_elements['txt_saatlik_ucret'].value = ""
    form_elements['txt_toplam_saat'].value = ""
    form_elements['txt_bayram_saat'].value = ""
    form_elements['btn_ekle_guncelle'].text = "1. Personeli Ekle ve Hesapla"
    form_elements['btn_ekle_guncelle'].bgcolor = RENK_PRİMER_MAVI
    page.update()

def guncelle_tablo_ve_ozet_action(form_elements: Dict, table_elements: Dict, page: ft.Page):
    """Hesaplamayı yeniden yapar, tabloyu ve özet satırını günceller."""
    
    df_rapor = hesapla_ve_raporla(STATE['personel_giris_listesi'], STATE['mevcut_kurallar'])
    
    table_elements['tablo_satirlari'].rows.clear()
    
    for index, row in df_rapor.iterrows():
        table_elements['tablo_satirlari'].rows.append(
            ft.DataRow(
                cells=[
                    ft.DataCell(ft.Text(row['Ad_Soyad'])),
                    ft.DataCell(ft.Text(f"{row['Normal_Mesai_Saat']:.1f}")),
                    ft.DataCell(ft.Text(f"{row['Fazla_Mesai_Saat']:.1f}")),
                    ft.DataCell(ft.Text(f"{row['Bayram_Calisma_Saat']:.1f}")),
                    ft.DataCell(ft.Text(f"{row['Normal_Mesai_Ucret']:.2f}")),
                    ft.DataCell(ft.Text(f"{row['Fazla_Mesai_Ucret']:.2f}")),
                    ft.DataCell(ft.Text(f"{row['Bayram_Mesai_Ucret']:.2f}")),
                    ft.DataCell(ft.Text(f"{row['Toplam_Ek_Odeme']:.2f}", weight=ft.FontWeight.BOLD)),
                    ft.DataCell(ft.Text(f"{row['Brüt_Maas_Tahmini']:.2f}", weight=ft.FontWeight.BOLD, color=RENK_PRİMER_MAVI)),
                    ft.DataCell(
                        ft.Row([
                            # Düzenle ve Sil butonları, ilgili aksiyonları çağırır
                            ft.TextButton("Düzenle", on_click=lambda e, i=index: duzenle_kayit_action(i, form_elements, table_elements, page), style=ft.ButtonStyle(color=RENK_DUZENLE)),
                            ft.TextButton("Sil", on_click=lambda e, i=index: silme_onay_action(i, form_elements, table_elements, page), style=ft.ButtonStyle(color=RENK_KIRMIZI_SIL)),
                        ], spacing=2)
                    ),
                ]
            )
        )

    genel_toplam = df_rapor['Toplam_Ek_Odeme'].sum() if not df_rapor.empty else 0.0
    table_elements['txt_genel_toplam'].value = f"GENEL TOPLAM (Ek Ödeme): {genel_toplam:.2f} TL"
    page.update()


# --- CRUD Eylemleri ---

def silme_onay_action(index: int, form_elements: Dict, table_elements: Dict, page: ft.Page):
    ad = STATE['personel_giris_listesi'][index]['Ad_Soyad']
    
    def silmeyi_onayla(e):
        STATE['personel_giris_listesi'].pop(index)
        # Eğer silinen kayıt o an düzenleniyorsa formu sıfırla
        if STATE['duzenlenen_kayit_index'] == index:
             reset_form_action(form_elements, page)
        
        guncelle_tablo_ve_ozet_action(form_elements, table_elements, page)
        page.close(dialog)
        page.snack_bar = ft.SnackBar(ft.Text(f"'{ad}' kaydı silindi."), duration=2000)
        page.snack_bar.open = True
        page.update()

    dialog = ft.AlertDialog(
        modal=True,
        title=ft.Text("Kayıt Silme Onayı", color=RENK_KIRMIZI_UYARI),
        content=ft.Text(f"'{ad}' adlı personelin puantaj kaydını kalıcı olarak silmek istediğinizden emin misiniz?"),
        actions=[
            ft.TextButton("İptal", on_click=lambda e: page.close(dialog)),
            ft.TextButton("Sil (Onayla)", on_click=silmeyi_onayla, style=ft.ButtonStyle(color=RENK_KIRMIZI_SIL)),
        ],
        actions_alignment=ft.MainAxisAlignment.END,
    )
    page.dialog = dialog
    dialog.open = True
    page.update()

def duzenle_kayit_action(index: int, form_elements: Dict, table_elements: Dict, page: ft.Page):
    kayit = STATE['personel_giris_listesi'][index]
    
    form_elements['txt_ad_soyad'].value = kayit['Ad_Soyad']
    form_elements['dd_kural_seti'].value = kayit['Kural_Seti']
    form_elements['txt_saatlik_ucret'].value = str(kayit['Saatlik_Ucret'])
    form_elements['txt_toplam_saat'].value = str(kayit['Toplam_Calisma_Saat'])
    form_elements['txt_bayram_saat'].value = str(kayit['Bayram_Calisma_Saat'])

    STATE['duzenlenen_kayit_index'] = index
    form_elements['btn_ekle_guncelle'].text = f"Kaydı Güncelle ({kayit['Ad_Soyad']})"
    form_elements['btn_ekle_guncelle'].bgcolor = RENK_DUZENLE
    page.update()

def ekle_ve_guncelle_action(e, form_elements: Dict, table_elements: Dict, page: ft.Page):
    try:
        ad = form_elements['txt_ad_soyad'].value
        saatlik_ucret = float(form_elements['txt_saatlik_ucret'].value or 0)
        toplam_saat = float(form_elements['txt_toplam_saat'].value or 0)
        bayram_saat = float(form_elements['txt_bayram_saat'].value or 0)
        kural_adi = form_elements['dd_kural_seti'].value
        
        if not kural_adi or not ad or saatlik_ucret < 0 or toplam_saat < 0 or bayram_saat < 0:
            raise ValueError("Tüm alanlar doldurulmalı ve geçerli değerler içermelidir.")

        yeni_veri = {
            'Ad_Soyad': ad, 'Kural_Seti': kural_adi, 'Saatlik_Ucret': saatlik_ucret,
            'Toplam_Calisma_Saat': toplam_saat, 'Bayram_Calisma_Saat': bayram_saat
        }
        
        if STATE['duzenlenen_kayit_index'] is not None:
            STATE['personel_giris_listesi'][STATE['duzenlenen_kayit_index']] = yeni_veri
            mesaj = f"'{ad}' kaydı güncellendi."
        else:
            STATE['personel_giris_listesi'].append(yeni_veri)
            mesaj = f"'{ad}' kaydı eklendi."
        
        guncelle_tablo_ve_ozet_action(form_elements, table_elements, page)
        reset_form_action(form_elements, page)
        page.snack_bar = ft.SnackBar(ft.Text(mesaj), duration=2000)
        page.snack_bar.open = True

    except ValueError as ve:
        page.snack_bar = ft.SnackBar(ft.Text(f"Giriş Hatası: {ve}"), duration=3000)
        page.snack_bar.open = True
    except Exception as ex:
        page.snack_bar = ft.SnackBar(ft.Text(f"Beklenmeyen Hata: {ex}"), duration=5000)
        page.snack_bar.open = True
    page.update()

def kayitlari_temizle_onay_action(e, form_elements: Dict, table_elements: Dict, page: ft.Page):
    def temizlemeyi_onayla(e):
        STATE['personel_giris_listesi'].clear()
        reset_form_action(form_elements, page)
        guncelle_tablo_ve_ozet_action(form_elements, table_elements, page)
        page.close(dialog)
        page.snack_bar = ft.SnackBar(ft.Text("Tüm puantaj kayıtları temizlendi."), duration=3000)
        page.snack_bar.open = True
        page.update()

    dialog = ft.AlertDialog(
        modal=True,
        title=ft.Text("Tüm Kayıtları Silme", color=RENK_KIRMIZI_UYARI),
        content=ft.Text(f"Tüm ({len(STATE['personel_giris_listesi'])}) personelin puantaj kaydını temizlemek istediğinizden emin misiniz? Bu işlem geri alınamaz!"),
        actions=[
            ft.TextButton("İptal", on_click=lambda e: page.close(dialog)),
            ft.TextButton("Tümünü Temizle (Onayla)", on_click=temizlemeyi_onayla, style=ft.ButtonStyle(color=RENK_KIRMIZI_SIL)),
        ],
        actions_alignment=ft.MainAxisAlignment.END,
    )
    page.dialog = dialog
    dialog.open = True
    page.update()

# --- Kural Yönetimi Aksiyonları ---

def kural_setlerini_guncelle_gui_action(kural_elements: Dict, form_elements: Dict, page: ft.Page):
    kural_elements['kural_listesi_view'].controls.clear()
    # Hem kural sekmesindeki hem de puantaj sekmesindeki dropdown'ı güncelle
    dropdowns_to_update = [kural_elements['dd_kural_seti_yonetim'], form_elements['dd_kural_seti']]
    
    for dd in dropdowns_to_update:
        dd.options.clear()

    mevcut_kurallar = STATE['mevcut_kurallar']

    if not mevcut_kurallar:
        kural_elements['kural_listesi_view'].controls.append(ft.Text("Tanımlı kural seti bulunmamaktadır."))
        for dd in dropdowns_to_update: dd.value = ""
    
    for ad, kural in mevcut_kurallar.items():
        for dd in dropdowns_to_update:
             dd.options.append(ft.dropdown.Option(ad))
        
        kural_elements['kural_listesi_view'].controls.append(
            ft.Card(
                content=ft.Container(
                    ft.Row(
                        [
                            ft.Column([
                                ft.Text(ad, weight=ft.FontWeight.BOLD, size=16),
                                ft.Text(f"Normal Sınır: {kural.NORMAL_AYLIK_SAAT} saat"),
                                ft.Text(f"FM Kat: x{kural.FAZLA_MESAI_KAT} | Bayram Kat: x{kural.BAYRAM_KAT}"),
                            ], spacing=5, expand=True),
                            
                            ft.TextButton(
                                "Sil", 
                                style=ft.ButtonStyle(color=RENK_KIRMIZI_SIL),
                                on_click=lambda e, kural_ad=ad: kural_silme_onay_action(e, kural_ad, kural_elements, form_elements, page)
                            )
                        ],
                        alignment=ft.MainAxisAlignment.SPACE_BETWEEN
                    ),
                    padding=10
                ),
                width=450
            )
        )
    
    # Dropdownlarda varsayılan değeri seç
    if mevcut_kurallar:
        first_kural = list(mevcut_kurallar.keys())[0]
        for dd in dropdowns_to_update:
            if not dd.value: dd.value = first_kural

    page.update()

def kural_silme_onay_action(e, kural_ad: str, kural_elements: Dict, form_elements: Dict, page: ft.Page):
    def silmeyi_onayla(e):
        STATE['mevcut_kurallar'].pop(kural_ad)
        kurallari_kaydet(STATE['mevcut_kurallar'])
        kural_setlerini_guncelle_gui_action(kural_elements, form_elements, page)
        page.close(dialog)
        page.snack_bar = ft.SnackBar(ft.Text(f"'{kural_ad}' kuralı silindi."), duration=2000)
        page.snack_bar.open = True
        page.update()

    dialog = ft.AlertDialog(
        modal=True,
        title=ft.Text("Onay Gerekiyor", color=RENK_KIRMIZI_UYARI),
        content=ft.Text(f"'{kural_ad}' kural setini kalıcı olarak silmek istediğinizden emin misiniz?"),
        actions=[
            ft.TextButton("İptal", on_click=lambda e: page.close(dialog)),
            ft.TextButton("Sil (Onayla)", on_click=silmeyi_onayla, style=ft.ButtonStyle(color=RENK_KIRMIZI_SIL)),
        ],
        actions_alignment=ft.MainAxisAlignment.END,
    )
    page.dialog = dialog
    dialog.open = True
    page.update()

def kural_ekle_action(e, kural_elements: Dict, form_elements: Dict, page: ft.Page):
    try:
        ad = kural_elements['txt_kural_ad'].value.strip()
        normal = float(kural_elements['txt_kural_normal_saat'].value)
        fm_kat = float(kural_elements['txt_kural_fm_kat'].value)
        bayram_kat = float(kural_elements['txt_kural_bayram_kat'].value)

        if not ad or normal <= 0 or fm_kat <= 0 or bayram_kat <= 0:
            raise ValueError("Tüm alanlar geçerli ve pozitif değerlerle doldurulmalıdır.")
        
        if ad in STATE['mevcut_kurallar']:
             raise ValueError("Bu kural adı zaten mevcut.")

        STATE['mevcut_kurallar'][ad] = KuralSeti(ad, normal, fm_kat, bayram_kat)
        kurallari_kaydet(STATE['mevcut_kurallar'])
        
        kural_elements['txt_kural_ad'].value = ""
        kural_setlerini_guncelle_gui_action(kural_elements, form_elements, page)
        page.snack_bar = ft.SnackBar(ft.Text(f"'{ad}' kuralı başarıyla eklendi."), duration=2000)
        page.snack_bar.open = True

    except ValueError as ve:
        page.snack_bar = ft.SnackBar(ft.Text(f"Kural Hatası: {ve}"), duration=3000)
        page.snack_bar.open = True
    page.update()

def raporu_indir_action(e, page: ft.Page):
    if not STATE['personel_giris_listesi']:
        page.snack_bar = ft.SnackBar(ft.Text("Kaydedilecek personel verisi yok!"), duration=2000)
        page.snack_bar.open = True
        page.update()
        return
        
    try:
        dosya_adi = f"puantaj_detay_raporu_{datetime.now().strftime('%Y%m%d_%H%M%S')}.xlsx"
        df_rapor = hesapla_ve_raporla(STATE['personel_giris_listesi'], STATE['mevcut_kurallar'])
        raporu_excel_kaydet(df_rapor, dosya_adi)
        
        page.snack_bar = ft.SnackBar(
            ft.Text(f"Excel Raporu başarıyla kaydedildi: {os.path.join(os.getcwd(), dosya_adi)}"),
            duration=5000
        )
    except Exception as ex:
        page.snack_bar = ft.SnackBar(ft.Text(f"Hata: Rapor kaydedilemedi! {ex}"), duration=5000)

    page.snack_bar.open = True
    page.update()

# ----------------------------------------------------------------------
# 3. ANA UYGULAMA YAPISI (main)
# ----------------------------------------------------------------------
def main(page: ft.Page):
    page.title = "PROFESYONEL Puantaj Yönetim Sistemi (Nihai Kararlı v5.3)"
    page.vertical_alignment = ft.MainAxisAlignment.START
    page.theme_mode = ft.ThemeMode.LIGHT
    page.window_height = 800
    page.window_width = 1650

    # --- BİLEŞENLERİN TANIMLANMASI VE GRUPLANMASI ---

    # PUANTAJ FORM ELEMANLARI
    form_elements = {}
    form_elements['txt_ad_soyad'] = ft.TextField(label="Personel Adı Soyadı", width=300)
    form_elements['txt_saatlik_ucret'] = ft.TextField(label="Saatlik Brüt Ücret (TL)", keyboard_type=ft.KeyboardType.NUMBER, width=300)
    form_elements['txt_toplam_saat'] = ft.TextField(label="Aylık Toplam Çalışma Saati", keyboard_type=ft.KeyboardType.NUMBER, width=300)
    form_elements['txt_bayram_saat'] = ft.TextField(label="Bayram Çalışma Saati", keyboard_type=ft.KeyboardType.NUMBER, width=300)
    form_elements['dd_kural_seti'] = ft.Dropdown(label="Kural Seti Seçimi (Puantaj)", options=[])
    # Buton tanımı en sonda yapılacak

    # PUANTAJ TABLO ELEMANLARI
    table_elements = {}
    table_elements['tablo_satirlari'] = ft.DataTable(
        columns=[
            ft.DataColumn(ft.Text("Ad Soyad")), ft.DataColumn(ft.Text("Normal Saat")), ft.DataColumn(ft.Text("FM Saat")),
            ft.DataColumn(ft.Text("Bayram Saat")), ft.DataColumn(ft.Text("Normal Ücret")), ft.DataColumn(ft.Text("FM Ücreti")),
            ft.DataColumn(ft.Text("Bayram Ücreti")), ft.DataColumn(ft.Text("EK ÖDEME", weight=ft.FontWeight.BOLD)),
            ft.DataColumn(ft.Text("BRÜT TAHMİNİ", weight=ft.FontWeight.BOLD, color=RENK_PRİMER_MAVI)),
            ft.DataColumn(ft.Text("Eylemler", weight=ft.FontWeight.BOLD)),
        ],
        rows=[], sort_column_index=0, sort_ascending=True, column_spacing=10
    )
    table_elements['txt_genel_toplam'] = ft.Text("GENEL TOPLAM (Ek Ödeme): 0.00 TL", size=20, weight=ft.FontWeight.BOLD, color=RENK_KIRMIZI_UYARI)

    # KURAL YÖNETİMİ ELEMANLARI
    kural_elements = {}
    kural_elements['txt_kural_ad'] = ft.TextField(label="Kural Adı (Örn: Beyaz Yaka)", width=300)
    kural_elements['txt_kural_normal_saat'] = ft.TextField(label="Aylık Normal Saat Sınırı (160.0)", keyboard_type=ft.KeyboardType.NUMBER, value="160.0", width=300)
    kural_elements['txt_kural_fm_kat'] = ft.TextField(label="Fazla Mesai Katsayısı (1.5)", keyboard_type=ft.KeyboardType.NUMBER, value="1.5", width=300)
    kural_elements['txt_kural_bayram_kat'] = ft.TextField(label="Bayram Katsayısı (2.0)", keyboard_type=ft.KeyboardType.NUMBER, value="2.0", width=300)
    kural_elements['kural_listesi_view'] = ft.Column(scroll=ft.ScrollMode.AUTO) 
    kural_elements['dd_kural_seti_yonetim'] = ft.Dropdown(label="Kural Seti Seçimi (Yönetim - Gizli)", options=[], visible=False) # Dummy dropdown for sync check

    # --- BUTONLARIN TANIMLANMASI (Artık elemanlar mevcut olduğu için lambda içinde gönderebiliriz) ---
    form_elements['btn_ekle_guncelle'] = ft.ElevatedButton(
        "1. Personeli Ekle ve Hesapla", 
        on_click=lambda e: ekle_ve_guncelle_action(e, form_elements, table_elements, page),
        bgcolor=RENK_PRİMER_MAVI, color=RENK_BEYAZ, width=300
    )

    btn_temizle = ft.TextButton(
        "Tüm Kayıtları Temizle",
        on_click=lambda e: kayitlari_temizle_onay_action(e, form_elements, table_elements, page),
        style=ft.ButtonStyle(color=RENK_KIRMIZI_SIL)
    )

    btn_kural_kaydet = ft.ElevatedButton(
        "Kural Setini Kaydet",
        on_click=lambda e: kural_ekle_action(e, kural_elements, form_elements, page),
        bgcolor=RENK_PRİMER_MAVI, color=RENK_BEYAZ, width=300
    )
    
    btn_rapor_indir = ft.ElevatedButton(
        "2. Excel Raporu OLUŞTUR", 
        on_click=lambda e: raporu_indir_action(e, page),
        bgcolor=RENK_YESIL_INDIR, color=RENK_BEYAZ,
    )


    # İlk çalıştırmada arayüzü güncelle
    kural_setlerini_guncelle_gui_action(kural_elements, form_elements, page)
    guncelle_tablo_ve_ozet_action(form_elements, table_elements, page)


    # --- ARAYÜZ YERLEŞİMİ (LAYOUT) ---
    
    # 1. Puantaj Hesaplama Sekmesi
    puantaj_hesaplama_icerigi = ft.Row(
        [
            ft.Container( # Sol Panel
                content=ft.Column(
                    [
                        ft.Text("Yeni Puantaj Girişi / Düzenleme", size=20, weight=ft.FontWeight.BOLD),
                        form_elements['txt_ad_soyad'], form_elements['dd_kural_seti'],
                        form_elements['txt_saatlik_ucret'], form_elements['txt_toplam_saat'], form_elements['txt_bayram_saat'],
                        form_elements['btn_ekle_guncelle'], btn_temizle
                    ], spacing=15
                ),
                padding=20, border_radius=10, bgcolor=RENK_GRI_ARKA_PLAN, width=350, alignment=ft.alignment.top_center
            ),
            ft.VerticalDivider(),
            ft.Container( # Sağ Panel
                content=ft.Column(
                    [
                        ft.Text("Detaylı Aylık Puantaj Dökümü", size=20, weight=ft.FontWeight.BOLD),
                        ft.Divider(),
                        ft.Container(content=ft.Column([table_elements['tablo_satirlari']], scroll=ft.ScrollMode.ADAPTIVE, expand=True), expand=True),
                        ft.Divider(height=2),
                        ft.Row([table_elements['txt_genel_toplam'], ft.Container(width=10), btn_rapor_indir], alignment=ft.MainAxisAlignment.SPACE_BETWEEN),
                    ], expand=True
                ),
                padding=20, expand=True,
            )
        ], expand=True, spacing=15, alignment=ft.MainAxisAlignment.START
    )

    # 2. Kural Yönetimi Sekmesi
    kural_yonetimi_icerigi = ft.Row([
        ft.Container( # Sol Panel
            content=ft.Column([
                ft.Text("Yeni Kural Seti Ekle", size=20, weight=ft.FontWeight.BOLD),
                kural_elements['txt_kural_ad'], kural_elements['txt_kural_normal_saat'], kural_elements['txt_kural_fm_kat'], kural_elements['txt_kural_bayram_kat'],
                btn_kural_kaydet
            ], spacing=15),
            padding=20, border_radius=10, bgcolor=RENK_GRI_ARKA_PLAN, width=350, alignment=ft.alignment.top_center
        ),
        ft.VerticalDivider(),
        ft.Container( # Sağ Panel
            content=ft.Column([
                ft.Text("Mevcut Tanımlı Kural Setleri", size=20, weight=ft.FontWeight.BOLD),
                ft.Divider(),
                kural_elements['kural_listesi_view']
            ], expand=True),
            padding=20, expand=True
        )
    ], expand=True, spacing=15, alignment=ft.MainAxisAlignment.START)

    # Ana Sayfaya Sekmeleri Ekle
    page.add(
        ft.Tabs(
            selected_index=0, animation_duration=300, expand=True,
            tabs=[
                ft.Tab(text="Puantaj Hesaplama", content=puantaj_hesaplama_icerigi),
                ft.Tab(text="Kural Yönetimi", content=kural_yonetimi_icerigi)
            ]
        )
    )
    page.update()

# Flet uygulamasını çalıştırma
if __name__ == "__main__":
    ft.app(target=main)
