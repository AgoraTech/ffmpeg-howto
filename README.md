O czym będzie
===============



Prerequisites
===============

Instalujemy z repozytoriów potrzebne biblioteki (wybieramy ich wersje -dev, czyli z plikami nagłówkowymi potrzebnymi do kompilacji zależnych od nich bibliotek, same biblioteki apt zainstaluje nam automatycznie).

    sudo apt-get -y install git autoconf automake build-essential libtool pkg-config texi2html zlib1g-dev

Tworzymy katalogi w którzych będziemy przeprowadzać instalację ffmpega:

    mkdir ffmpeg_sources
    mkdir ffmpeg_build
    cd ffmpeg_sources/

Ponieważ nasz ffmpeg i kodeki korzystają z bardzo szybkich optymalizacji w assemblerze, potrzebny nam będzie również relatywnie nowy `YASM`. W repozytoriach niestety występuje bardzo stara wersja, więc instalujemy najnowszą ze źródeł pobranych ze strony: http://yasm.tortall.net

    wget http://www.tortall.net/projects/yasm/releases/yasm-1.2.0.tar.gz -qO - | tar zx
    cd yasm-1.2.0/
    ./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin"
    time ( make -j 3 && make install )
    cd ..

Aby system zauważył, że YASM jest już dostępny musimy ponownie załadować profil użytkownika (uruchomić skrypt shellowy uruchamiany po zalogowaniu użytkownika).

    . ~/.profile

Instalacja programów enkodujących
===================================

W tym kroku zainstalujemy kodeki i ffmpeg, czyli najbardziej znany program do kodowania materiałów wideo, zacznijmy od kodeków, które są na tyle stabilne (w rozwoju), że możemy pozwolić sobie na instalację z repozytoriów, czyli `vorbis` oraz `theora`.

    sudo apt-get -y install autoconf libtheora-dev libvorbis-dev libass-dev libgpac-dev libsdl1.2-dev libva-dev libvdpau-dev libx11-dev libxext-dev libxfixes-dev libmp3lame-dev

Dodatkowo instalujemy akcelerację sprzętową VAAPI i VDPAU, pozwoli to szybciej dekodować wideo.

H.264 i AAC
-------------

Najpowszechniejsze w chwili obecnej kodeki video i audio to odpowiednio H.264 i AAC. Choć są to kodeki o zamkniętej licencji, istnieją ich opensource'owe implementacje, z których obecnie najlepszymi są:

H.264 - x264 by VideoLAN - kodek od firmy, która stworzyła najlepszy otwarty odtwarzacz multimediów, VLC.

    git clone --depth 1 git://git.videolan.org/x264.git
    cd x264/
    ./configure --prefix="$HOME/ffmpeg_build" --bindir="$HOME/bin" --enable-static
    time ( make -j 3 && make install )
    cd ..

AAC - FDK-AAC - implementacja kodeka AAC dostarczona przez instytut Frauenhoffera, będąca częścią projektu Android.

    git clone --depth 1 git://github.com/mstorsjo/fdk-aac.git
    cd fdk-aac/
    autoreconf -fiv
    ./configure --prefix="$HOME/ffmpeg_build" --disable-shared
    time ( make -j 3 && make install )
    cd ..

Inne kodeki
-------------

VP9 - otwarty i wolnościowy kodek video stworzony przez jedeną z firm zależnych Google, On2 Technologies.

    git clone --depth 1 http://git.chromium.org/webm/libvpx.git
    cd libvpx/
    ./configure --prefix="$HOME/ffmpeg_build" --disable-examples
    time ( make -j 3 && make install )
    cd ..

MP3 - znany chyba wszystkim kodek audio Mpeg Layer-3.

    wget http://downloads.sourceforge.net/project/lame/lame/3.99/lame-3.99.5.tar.gz -qO - | tar xz
    cd lame-3.99.5
    ./configure --prefix="$HOME/ffmpeg_build" --disable-shared
    time ( make -j 3 && make install )
    cd ..

FFMpeg
--------

FFMpeg to kombajn do przekodowywania filmów z jednego formatu na drugi, trzeciego na czwarty, dokładania piątego filmu w tle, łączenia z szóstym, dogrywania ścieżki audio z siódmego, dodania obrazka w górnym lewym rogu ekranu... i tak dalej...

    git clone --depth 1 git://source.ffmpeg.org/ffmpeg
    cd ffmpeg/
    PKG_CONFIG_PATH="$HOME/ffmpeg_build/lib/pkgconfig"
    export PKG_CONFIG_PATH
    ./configure --prefix="$HOME/ffmpeg_build" --extra-cflags="-I$HOME/ffmpeg_build/include" --extra-ldflags="-L$HOME/ffmpeg_build/lib"   --bindir="$HOME/bin" --extra-libs="-ldl" --enable-gpl --enable-libass --enable-libfdk-aac --enable-libmp3lame --enable-libtheora --enable-libvorbis --enable-libvpx   --enable-libx264 --enable-nonfree
    time ( make -j 3 && make install )
    cd ..

Przekodujmy materiał
======================

Po skompilowaniu wszystkich narzędzi możemy przejść do kodowania naszego materiału - zakodujemy od razu dwie jakości, jedną w wysokiej rozdzielczości, drugą w niższej, dla słabszych łącz. Pisząc aplikację musicie uwzględnić dwie możliwości, dając użytkownikowi dodatkowy przycisk np. HD lub HQ.

Nagrałem film z poprzedniej prezentacji Kamila, będzie on naszym źródłem.

Wszystkie filmy przekodujemy do katalogu /g/video5, musimy go założyć `sudo mkdir -p /g/video5 && sudo chown -R michal /g`

Identyfikacja
---------------

Najpierw sprawdźmy co wiemy o tym filmie:

    ffprobe <źródło>

W wyniku mamy na przykład:

    Input #0, mov,mp4,m4a,3gp,3g2,mj2, from '/media/sf_share/20131118_142416.mp4':
      Metadata:
        major_brand     : isom
        minor_version   : 0
        compatible_brands: isom3gp4
        creation_time   : 2013-11-18 13:30:55
      Duration: 00:06:38.10, start: 0.000000, bitrate: 17106 kb/s
        Stream #0:0(eng): Video: h264 (High) (avc1 / 0x31637661), yuv420p, 1920x1080, 16987 kb/s, 29.98 fps, 30 tbr, 90k tbn, 180k tbc (default)
        Metadata:
          creation_time   : 2013-11-18 13:30:55
          handler_name    : VideoHandle
        Stream #0:1(eng): Audio: aac (mp4a / 0x6134706D), 48000 Hz, stereo, fltp, 123 kb/s (default)
        Metadata:
          creation_time   : 2013-11-18 13:30:55
          handler_name    : SoundHandle

Kodowanie
-----------

Linia poleceń ffmpeg składać się będzie z następujących poleceń:

* źródło to nasz film `-i <film_wejściowy>`
* na moich maszynach mamy do dyspozycji 4 procesory, uruchamiamy więc `-threads 4`
* po podaniu kodeka wideo rozpoczynamy listę opcji streamu video
  - użyjemy kodeka H.264 za pomocą skompilowanego wcześniej `-codec:v libx264`
  - chcemy, żeby nasza wynikowa rozdzielczość wyniosła `-s 1280x720`, dając proporcje 16x9
  - wskażemy oczekiwany poziom jakości kodowania za pomocą `-q 5`
  - podamy preset, czyli jak bardzo x264 ma starać się zachować jakość, powiedzmy, że medium `-preset medium`
  - podamy interesujący nas bitrate wideo `-b:v 1536k` - oznacza to, że każda sekunda zajmie przeciętnie 1536k bitów, co daje 192 kB na samo wideo.
  - chcemy, aby klatka kluczowa była zawarta raz na conajmniej 4 sekundy `-g 120` i nie częściej niż co 5 klatek `-keyint_min 5`
* następnie deniniujemy audio:
  - użyjmy przed chwilą skompilowanego kodeka AAC `-codec:a libfdk_aac`
  - chcemy zakodować dwa kanały (stereo) `-ac 2`
  - nasz dźwięk ma mieć próbkowanie 44.1kHz, czyli analogiczne do spotykanego na płytach CD `-ar 44100`
  - w formacie AAC straty przy bitrate `-b:a 128k` są nieznaczne.
* chcemy, aby nasz film miał informację o wszystkich klatkach kluczowych na początku pliku, a nie tak jak domyślnie na końcu `-movflags faststart`.
* wskazujemy dokąd plik ma być kodowany `/g/video5/spotkanie_720p.mp4`
* wszystko co podamy dalej spowoduje zakodowanie drugiego pliku wynikowego, dla nas: `-codec:v libx264 -s 640x360 -q 5 -preset medium -g 120 -keyint_min 5 -b:v 700k -codec:a libfdk_aac -ac 2 -ar 44100 -b:a 64k -movflags faststart public/spotkanie_360p.mp4`
* dodatkowo chcemy wyeksportować obrazek, więc na końcu dodajmy kolejne opcje:
  - nie potrzebujemy audio, więc `-an`
  - kodek `-codec:v mjpeg` tworzy po prostu po jednym obrazku w formacie jpeg na każdą klatkę filmu
  - chcemy wyciągnąć obrazek z 10tej sekundy filmy, więc `-ss 0:0:10.0`
  - oczywiście interesuje nas jedna klatka, więc ustawiamy kombinację czasu i ilości klatek na sekundę odpowiednio `-t 1 -r 1`.
  - interesują nas jeszcze rozmiary tego obrazka, powiedzmy `-s 320x180`
  - nazwiemy go `/g/video5/spotkanie_preview.jpg`

Nasza linia poleceń będzie wyglądać następująco:

    ffmpeg -i trailer_1080p.ogg -threads 4 -codec:v libx264 -s 1280x720 -q 5 -preset medium -b:v 1536k -g 120 -keyint_min 5 -codec:a libfdk_aac -ac 2 -ar 44100 -b:a 128k -movflags faststart /g/video5/spotkanie_720p.mp4 -codec:v libx264 -s 640x360 -q 5 -preset medium -g 120 -keyint_min 5 -b:v 700k -codec:a libfdk_aac -ac 2 -ar 44100 -b:a 64k -movflags faststart /g/video5/spotkanie_360p.mp4 -an -codec:v mjpeg -ss 0:0:10.0 -t 1 -r 1 -s 320x180 /g/video5/spotkanie_preview.jpg

Apache
========

Pozostaje nam skonfigurowanie apache - to najpopularniejszy serwer HTTP i choć nie najprostszy, na pewno na 99% serwerów na których będziecie pracować będzie on domyślnie zainstalowany.

Oczywiście, każdy wybiera swoje własne weapon of choice.

        Alias /video5 /g/video5
        <Directory /g/video5>
                Options -Indexes
                AllowOverride none
                Order allow,deny
                allow from all
        <Directory>

Zdefiniowaliśmy alias do wirtualnego katalogu na serwerze na nasz dysk i już możemy oglądać wideo, choćby pod adresem: `http://mydomain.com/video5/spotkanie_360p.mp4`.
