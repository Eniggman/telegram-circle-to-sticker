---
name: telegram-circle-to-sticker
description: "Convert a Telegram round video message ('кружочек') or any square/short video into a Telegram video sticker: WebM VP9 with a transparent circular alpha mask, 512x512, 30 fps, max 3 s, max 256 KB, no audio, using FFmpeg + a small Pillow script. Use when the user wants to turn a video note, clip or GIF-like video into a round video sticker for @Stickers, or their sticker has white/black corners. Triggers: «кружок в стикер», «кружочек в стикер», «видеостикер из кружка», «сделай стикер из видео», «webm стикер», «прозрачные углы стикера», «Telegram video sticker»."
---

# Telegram: кружочек → видеостикер WebM VP9 с прозрачной маской

Инструкция для ИИ-агента. Для людей: [README](https://github.com/Eniggman/telegram-circle-to-sticker#readme). Официальные требования Telegram: https://core.telegram.org/stickers/webm-vp9-encoding

## Требования к результату

- Контейнер `.webm`, кодек VP9 (`libvpx-vp9`), пиксельный формат `yuva420p` с альфа-каналом.
- Размер 512×512 (одна сторона ровно 512), не больше 30 FPS, **не больше 3,0 с**, без аудио.
- Файл не больше 256 КБ. Цель — **≤ 256 000 байт** с запасом 3–6 КБ. README пишет 262 144 байта, но консервативный предел подходит под оба прочтения.
- Углы прозрачные (alpha 0), центр непрозрачный (alpha 255).

## Шаг 0. Окружение

```bash
ffmpeg -hide_banner -encoders | grep libvpx-vp9   # должен быть libvpx-vp9
python3 -c "import PIL; print(PIL.__version__)"     # Pillow
```
Если чего-то не хватает, **спроси разрешения** на установку: `sudo apt install -y ffmpeg python3-pil` (Debian/Ubuntu) или `pip install pillow`.
Скрипт масок: `scripts/make_circle_frames.py` из репозитория (`git clone https://github.com/Eniggman/telegram-circle-to-sticker`).

## Шаг 1. Получить исходник

Попроси у человека путь к файлу. Кружочек из Telegram Desktop сохраняется через ПКМ → «Сохранить как» и обычно оказывается квадратным MP4: круглым его делает только клиент. Проверь исходник:
```bash
ffprobe -v error -show_entries stream=codec_type,codec_name,width,height,r_frame_rate:format=duration -of compact input.mp4
```
Если ролик длиннее 3 с, **спроси, какой фрагмент взять** (по умолчанию первые 3 с: `-ss <начало> -t 3`).

## Шаг 2. Рекомендуемый пайплайн (проверен)

```bash
mkdir -p input_frames rgba_frames
# 1) обрезка до 3 с, кадрирование по центру до 512x512, 30 fps, PNG-кадры
ffmpeg -y -i input.mp4 -t 3 \
  -vf "fps=30,scale=512:512:force_original_aspect_ratio=increase,crop=512:512" \
  input_frames/%04d.png
# 2) круглая альфа-маска (эллипс от (0,0) до (511,511))
python3 scripts/make_circle_frames.py input_frames rgba_frames
# 3) кодирование в VP9 с альфой
ffmpeg -y -framerate 30 -i 'rgba_frames/%04d.png' -an \
  -vf 'format=yuva420p' -c:v libvpx-vp9 -pix_fmt yuva420p \
  -b:v 200k -crf 48 -deadline good -cpu-used 2 -row-mt 1 \
  -auto-alt-ref 0 -metadata:s:v:0 alpha_mode=1 \
  sticker.webm
```
`-auto-alt-ref 0` обязателен: без него VP9 портит прозрачность. Углы не заливай цветом: нужна именно прозрачность.

Альтернатива одной командой, без промежуточных PNG:
```bash
ffmpeg -y -i input.mp4 -t 3 -an \
  -vf "fps=30,scale=512:512:force_original_aspect_ratio=increase,crop=512:512,format=yuva420p,geq=lum='p(X,Y)':a='if(lte(pow(X-255.5,2)+pow(Y-255.5,2),pow(255.5,2)),255,0)'" \
  -c:v libvpx-vp9 -pix_fmt yuva420p -b:v 200k -crf 48 \
  -deadline good -cpu-used 2 -row-mt 1 -auto-alt-ref 0 \
  -metadata:s:v:0 alpha_mode=1 sticker.webm
```
Двухэтапный вариант даёт более ровный край круга, поэтому по умолчанию используй его.

## Шаг 3. Уложиться в размер

```bash
stat -c %s sticker.webm        # Linux; macOS: stat -f %z; Windows PowerShell: (Get-Item sticker.webm).Length
```
Если больше 256 000 байт, перекодируй шаг 3 и меняй по одному параметру: `-crf` вверх (48 → 52 → 56, максимум 63) или `-b:v` вниз (200k → 150k → 100k). Последнее средство — сократить длительность. Каждый раз заново проверяй размер.

## Шаг 4. Проверка результата

```bash
ffprobe -v error -show_entries stream=codec_name,width,height,r_frame_rate:format=duration,size -of compact sticker.webm
# ожидается: codec_name=vp9, 512x512, r_frame_rate=30/1, duration<=3.0
ffprobe -v error -select_streams a -show_entries stream=index -of csv=p=0 sticker.webm   # пусто = аудио нет
# альфу проверяй ТОЛЬКО через декодер libvpx: обычный ffprobe/декодер покажет yuv420p и непрозрачные углы
ffmpeg -y -c:v libvpx-vp9 -i sticker.webm -frames:v 1 -pix_fmt rgba check.png
python3 -c "from PIL import Image; im=Image.open('check.png'); print('corner', im.getpixel((0,0))[3], 'center', im.getpixel((256,256))[3])"
# ожидается: corner 0 center 255
```
После проверки удали временные файлы: `rm -rf input_frames rgba_frames check.png`.

## Шаг 5. Передать человеку

Отдай `sticker.webm` и скажи, что добавить его в набор можно через официального бота `@Stickers` (это делает человек сам в Telegram).

## Неофициально: ролик длиннее 3 секунд (spoofing)

Только если человек **явно** просит полный ролик. Предупреди: это подмена поля длительности в контейнере WebM, а не снятие лимита. `@Stickers` или мобильные клиенты могут отклонить файл или проиграть его неправильно. Пакет `tgradish` сторонний (https://github.com/sliva0/tgradish, автор sliva0): перед установкой спроси разрешения.
```bash
pip install tgradish
python3 -m tgradish spoof input_long.webm sticker_spoof.webm
```
Ограничение по размеру (256 КБ) остаётся.

## Частые ошибки

| Симптом | Причина и решение |
|---|---|
| Белые или чёрные углы у стикера | Нет маски, нет `-auto-alt-ref 0`, нет `alpha_mode=1` или `-pix_fmt yuva420p` |
| Проверка показывает непрозрачные углы | Декодировали без `-c:v libvpx-vp9`: повтори проверку из шага 4 |
| `@Stickers` отклоняет файл | Длительность больше 3 с, размер больше 256 КБ, есть аудио, FPS больше 30 или сторона не 512 |
| `No PNG frames found in input_frames` | Шаг 1 не создал кадры: проверь путь к `input.mp4` и вывод ffmpeg |
| `Unknown encoder 'libvpx-vp9'` | Сборка FFmpeg без VP9: нужна полная сборка FFmpeg |
| Голова в кадре обрезана | Кадрирование по центру. Предложи человеку сдвинуть кроп: `crop=512:512:x:y` вместо `crop=512:512` |
