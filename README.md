# Porównywarka podpisów APK

Jednoplikowa aplikacja webowa (czysty JavaScript, bez bibliotek i bez serwera), która odczytuje podpisy plików APK, porównuje certyfikaty dwóch plików i sprawdza, czy plik nie został zmieniony po podpisaniu. Analiza odbywa się w całości w przeglądarce – pliki nie są nigdzie wysyłane.

## Do czego służy

Masz plik APK ze znanego źródła (np. aplikację z własnego telefonu) i drugi, pobrany z Internetu. Aplikacja odpowiada na pytania:

1. Kto widnieje w certyfikacie podpisu (podmiot, wystawca, ważność, numer seryjny, typ klucza).
2. Jaki jest odcisk SHA-256 (i SHA-1) certyfikatu.
3. Czy podpis jest poprawny, a zawartość pliku nie została zmodyfikowana po podpisaniu.
4. Czy oba pliki są podpisane tym samym kluczem.
5. Czy certyfikat zgadza się ze znanym odciskiem (wpisanym, wybranym z listy lub wbudowanym).

## Tryby pracy

- **Dwa pliki (A i B).** A to plik zaufany, B – do sprawdzenia. Wynik zawiera werdykt porównania, a pod nim karty obu plików.
- **Jeden plik.** Powstaje jedna karta na całą szerokość. Plik można dodatkowo sprawdzić ze znanym odciskiem (wpisanym lub wybranym z listy).
- **Paczki.** Pliki `.apk+` (kopie zapasowe CX File Explorera), `.apks`, `.xapk` i `.apkm` to archiwa ZIP z wieloma APK. Każdy APK z paczki dostaje własną kartę i własne linie w wyniku. Split z innym certyfikatem niż `base.apk` jest oznaczany alarmem. Dane z `apk+.json` (nazwa aplikacji, pakiet, wersja) są pokazywane jako uwaga.

## Co jest sprawdzane

| Schemat | Odczyt certyfikatu | Weryfikacja integralności |
|---|---|---|
| v1 (JAR, `META-INF/*.RSA/.DSA/.EC`) | tak | podpis pliku `.SF`, skrót `MANIFEST.MF` (lub skróty sekcji), skróty wszystkich plików, wykrywanie plików dodanych po podpisaniu |
| v2 | tak | podpis kryptograficzny (RSA, RSA-PSS, ECDSA) oraz skrót zawartości całego APK (bloki 1 MB) |
| v3 / v3.1 | tak | jak v2; dodatkowo odczyt historii rotacji klucza |

Rozpoznawanie producentów: bieżący certyfikat (oraz starsze z historii rotacji) jest porównywany z wbudowaną listą. Lista zawiera certyfikat Samsunga (Samsung Cert), certyfikat Google (CN=Android, O=Google Inc.) i publiczne klucze testowe AOSP (testkey, platform, shared, media). Wpisy mają klikalne źródła. Są to źródła nieoficjalne (APKMirror, zgłoszenia na GitHubie, wiki projektu AV) – producenci zwykle nie publikują odcisków. Klucze testowe AOSP są publiczne, więc ich rozpoznanie jest ostrzeżeniem, a nie dowodem pochodzenia.

## Jak używać

1. Otwórz stronę aplikacji.
2. Przeciągnij pliki na pola A i B albo kliknij pole i wybierz plik. Na komputerze lista rozszerzeń w oknie wyboru to `.apk, .apk+, .apks, .xapk, .apkm, .aab, .zip`, na urządzeniach mobilnych filtra nie ma.
3. Opcjonalnie wpisz znany odcisk SHA-256 certyfikatu producenta albo wybierz go z listy.
4. Kliknij „Analizuj i porównaj”.

Interfejs ma przełącznik języka (PL/EN) i motywu (jasny/ciemny). Wybór jest zapamiętywany w przeglądarce; domyślnie stosowane są ustawienia przeglądarki i systemu.

## Jak czytać wynik

- **Zielone „OK”** – sprawdzenie przeszło (np. ten sam certyfikat, brak modyfikacji po podpisaniu).
- **Żółte „Uwaga”** – czegoś nie da się ustalić (np. nieobsługiwany algorytm) albo jest to ostrzeżenie (certyfikat debug, klucz testowy AOSP).
- **Czerwone „Alarm”** – niezgodność: inny certyfikat, błędny podpis, zmodyfikowany lub dodany plik.

## Ograniczenia

- Certyfikat nie mówi sam, do jakiej firmy należy. Pole „Podmiot” może wpisać każdy. Pochodzenie ustala się dopiero przez zgodność odcisku ze znanym plikiem lub z odciskiem opublikowanym przez producenta. Najpewniejszym wzorcem jest oryginalny plik z własnego urządzenia.
- Kontrola „modyfikacji po podpisaniu” wykrywa zmiany bez ponownego podpisania. APK przepakowany i podpisany nowym kluczem ma poprawny podpis, ale inny certyfikat.
- Historia rotacji klucza jest odczytywana heurystycznie, a podpisy w łańcuchu nie są weryfikowane.
- Nieobsługiwane: podpis DSA, RSA-PSS w v1, ZIP64.
- Brak rozpoznania certyfikatu nie oznacza, że plik jest fałszywy, a rozpoznanie nie dowodzi, że jest autentyczny.
- Duże pliki analizowane są w pamięci przeglądarki; analiza może chwilę potrwać.

## Wymagania techniczne

Nowoczesna przeglądarka z WebCrypto (`crypto.subtle`, wymaga HTTPS lub localhost) i `DecompressionStream` (`deflate-raw`). Zewnętrzne biblioteki nie są używane.

## Struktura kodu (jeden plik HTML)

Kod jest opisany komentarzami JSDoc. Główne części:

- **Słownik tłumaczeń** (`L`, `t`, `M`, `tx`) – komunikaty analizy są niezależne od języka (`M`) i tłumaczone dopiero przy wyświetlaniu, dzięki czemu zmiana języka działa na gotowych wynikach.
- **DER / X.509** (`tlv`, `kids`, `oid`, `dname`, `dtime`, `parseCert`, `derToRaw`) – minimalny parser certyfikatów.
- **Analiza APK** (`analyze` i funkcje wewnętrzne) – odczyt ZIP, podpisy v1 (`verifyV1`), v2/v3 (`content`, `verify`), historia rotacji (`readLineage`), obsługa paczek.
- **Znane certyfikaty** (`KNOWN`, `SRCURL`, `findKnown`, `linkify`).
- **Interfejs** (`bind`, `card`, `fileLines`, `linLines`, `expLines`, `cardsOf`, `render`, `applyLang`, `applyTheme`, obsługa przycisku analizy).

Typy wyników (`Cert`, `Signer`, `AnalysisResult`, `Msg`, `Verdict`) opisane są w nagłówku JSDoc na początku skryptu.
