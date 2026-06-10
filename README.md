// Vercel Serverless Function — генератор ICS для марафона Шамиля Ахмадуллина
// GET /api/calendar        — все 5 дней
// GET /api/calendar?day=3  — только день 3

const MARATHON_NAME = 'Марафон Шамиля Ахмадуллина';
const MARATHON_START_DAYS = [1, 4]; // 1=пн, 4=чт

const DAYS_DATA = [
  {
    summary: 'Телефон важнее мамы и учёбы? Разбираем на эфире',
    description: 'Что делать, если все разговоры и запреты не работают. Разбираем вместе на вебинаре:\\nhttps://lesson.shamilahmadullin.com/calendar_d1',
    location: 'https://lesson.shamilahmadullin.com/calendar_d1',
    alarms: [
      { offset: -1440, text: '⏰ Завтра в 19:00 МСК — эфир про гаджеты и детей. Не пропустите 👉 https://lesson.shamilahmadullin.com/calendar_d1' },
      { offset: -60,   text: '🔔 Через час разберём: почему запреты не работают — и что работает 👉 https://lesson.shamilahmadullin.com/calendar_d1' },
      { offset: -10,   text: '🚀 Через 10 минут всё станет понятно. Открывайте уже сейчас 👉 https://lesson.shamilahmadullin.com/calendar_d1' },
    ],
  },
  {
    summary: 'Не хочет учиться — как это лечится: узнайте на эфире',
    description: 'Откуда берётся апатия к учёбе и как помочь ребёнку вернуть желание — разбираем в прямом эфире:\\nhttps://lesson.shamilahmadullin.com/calendar_d2',
    location: 'https://lesson.shamilahmadullin.com/calendar_d2',
    alarms: [
      { offset: -60, text: '🔔 Через час — про мотивацию к учёбе. Не пропустите 👉 https://lesson.shamilahmadullin.com/calendar_d2' },
      { offset: -10, text: '🚀 Через 10 минут узнаете, как вернуть ребёнку желание учиться 👉 https://lesson.shamilahmadullin.com/calendar_d2' },
    ],
  },
  {
    summary: 'Умный, но не учится — главный эфир марафона',
    description: 'Почему ребёнок не запоминает и не может сосредоточиться? Разбираемся на главном вебинаре марафона:\\nhttps://lesson.shamilahmadullin.com/calendar_d3',
    location: 'https://lesson.shamilahmadullin.com/calendar_d3',
    alarms: [
      { offset: -60, text: '🔔 Главный эфир марафона — через час. Не пропустите 👉 https://lesson.shamilahmadullin.com/calendar_d3' },
      { offset: -10, text: '🚀 Через 10 минут — главный эфир. Уже открывайте ссылку 👉 https://lesson.shamilahmadullin.com/calendar_d3' },
    ],
  },
  {
    summary: 'Почему ребёнок слушает всех, кроме вас?',
    description: 'Как говорить с ребёнком так, чтобы он слышал — без скандалов и повторений:\\nhttps://lesson.shamilahmadullin.com/calendar_d4',
    location: 'https://lesson.shamilahmadullin.com/calendar_d4',
    alarms: [
      { offset: -60, text: '🔔 Через час — про общение с детьми без конфликтов 👉 https://lesson.shamilahmadullin.com/calendar_d4' },
      { offset: -10, text: '🚀 Через 10 минут узнаете, почему ребёнок не слышит — и как это исправить 👉 https://lesson.shamilahmadullin.com/calendar_d4' },
    ],
  },
  {
    summary: 'Как вырастить самостоятельного ребёнка — финальный эфир',
    description: 'Финал марафона. Раскрываем тайну «как воспитать ответственного и самостоятельного ребёнка» в прямом эфире:\\nhttps://lesson.shamilahmadullin.com/calendar_d5',
    location: 'https://lesson.shamilahmadullin.com/calendar_d5',
    alarms: [
      { offset: -60, text: '🔔 Финальный эфир через час — сегодня самое важное 👉 https://lesson.shamilahmadullin.com/calendar_d5' },
      { offset: -10, text: '🚀 Через 10 минут — финал марафона. Не опаздывайте 👉 https://lesson.shamilahmadullin.com/calendar_d5' },
    ],
  },
];

function pad(n) { return String(n).padStart(2, '0'); }

function icsDate(date) {
  return date.getUTCFullYear() + pad(date.getUTCMonth() + 1) + pad(date.getUTCDate()) +
    'T' + pad(date.getUTCHours()) + pad(date.getUTCMinutes()) + '00Z';
}

function makeTrigger(offsetMinutes) {
  if (offsetMinutes === 0) return 'TRIGGER:PT0S';
  const sign = offsetMinutes < 0 ? '-' : '+';
  const abs = Math.abs(offsetMinutes);
  const d = Math.floor(abs / 1440);
  const h = Math.floor((abs % 1440) / 60);
  const m = abs % 60;
  if (d > 0) return `TRIGGER:${sign}P${d}DT${pad(h)}H${pad(m)}M0S`;
  return `TRIGGER:${sign}P0DT${pad(h)}H${pad(m)}M0S`;
}

function makeAlarmBlock(alarm) {
  return ['BEGIN:VALARM', 'ACTION:DISPLAY', `DESCRIPTION:${alarm.text}`, makeTrigger(alarm.offset), 'END:VALARM'].join('\r\n');
}

// МСК = UTC+3. Если сегодня пн/чт И до 19:00 МСК — стартуем сегодня.
// Иначе — ближайший следующий пн или чт.
function findNextStart(now) {
  const mskNow = new Date(now.getTime() + 3 * 60 * 60 * 1000);
  const todayUTC = new Date(Date.UTC(mskNow.getUTCFullYear(), mskNow.getUTCMonth(), mskNow.getUTCDate()));
  const todayEventUTC = new Date(todayUTC.getTime() + 16 * 60 * 60 * 1000); // 19:00 МСК = 16:00 UTC

  if (MARATHON_START_DAYS.includes(todayUTC.getUTCDay()) && now < todayEventUTC) {
    return todayUTC;
  }
  for (let i = 1; i <= 7; i++) {
    const candidate = new Date(todayUTC.getTime() + i * 86400000);
    if (MARATHON_START_DAYS.includes(candidate.getUTCDay())) return candidate;
  }
  return todayUTC;
}

function buildICS(startDate, now, dayIndex = null) {
  const lines = [
    'BEGIN:VCALENDAR',
    'VERSION:2.0',
    'PRODID:-//Shamil Ahmadullin//Marathon//RU',
    'CALSCALE:GREGORIAN',
    'METHOD:PUBLISH',
    `X-WR-CALNAME:${MARATHON_NAME}`,
    'X-WR-TIMEZONE:Europe/Moscow',
  ];

  const indices = dayIndex !== null ? [dayIndex] : [0, 1, 2, 3, 4];

  for (const i of indices) {
    const day = DAYS_DATA[i];
    const eventDate = new Date(startDate.getTime() + i * 86400000);
    const dtStart = new Date(eventDate.getTime() + 16 * 60 * 60 * 1000); // 19:00 МСК

    lines.push(
      'BEGIN:VEVENT',
      `UID:marathon-d${i + 1}-${now.getTime()}@shamil`,
      `DTSTAMP:${icsDate(now)}`,
      `DTSTART:${icsDate(dtStart)}`,
      `SUMMARY:${day.summary}`,
      `DESCRIPTION:${day.description}`,
      `LOCATION:${day.location}`,
      'TRANSP:OPAQUE',
      'STATUS:CONFIRMED',
      ...day.alarms.map(makeAlarmBlock),
      'END:VEVENT'
    );
  }

  lines.push('END:VCALENDAR');
  return lines.join('\r\n');
}

export default function handler(req, res) {
  if (req.method !== 'GET') {
    return res.status(405).send('Method Not Allowed');
  }

  const now = new Date();
  const startDate = findNextStart(now);

  let dayIndex = null;
  const dayParam = req.query.day;
  if (dayParam !== undefined) {
    const d = parseInt(dayParam, 10);
    if (isNaN(d) || d < 1 || d > 5) {
      return res.status(400).send('Параметр day должен быть от 1 до 5');
    }
    dayIndex = d - 1;
  }

  const icsContent = buildICS(startDate, now, dayIndex);

  const startStr = startDate.getUTCFullYear() +
    pad(startDate.getUTCMonth() + 1) +
    pad(startDate.getUTCDate());
  const filename = dayIndex !== null
    ? `marathon-day${dayIndex + 1}-${startStr}.ics`
    : `marathon-${startStr}.ics`;

  res.setHeader('Content-Type', 'text/calendar; charset=utf-8');
  res.setHeader('Content-Disposition', `attachment; filename="${filename}"`);
  res.setHeader('Cache-Control', 'no-cache, no-store, must-revalidate');
  res.setHeader('Access-Control-Allow-Origin', '*');
  res.status(200).send(icsContent);
}
