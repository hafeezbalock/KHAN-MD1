import makeWASocket, { 
    useMultiFileAuthState, 
    DisconnectReason, 
    Browsers,
    fetchLatestBaileysVersion 
} from '@whiskeysockets/baileys';

import pino from 'pino';
import QRCode from 'qrcode';
import qrcodeTerminal from 'qrcode-terminal';
import express from 'express';
import { google } from 'googleapis';
import { GoogleGenerativeAI } from '@google/generative-ai';
import { Jimp } from 'jimp';
import fs from 'fs';

// --------------------
// WEB SERVER (PORT 5000)
// --------------------
const app = express();
let latestQR = null;

app.get('/', (req, res) => res.send('🏥 Hospital Bot is Live!'));
app.get('/qr', async (req, res) => {
    if (!latestQR) return res.send('<h2>QR scanned or not ready. Check Console.</h2>');
    const qrImage = await QRCode.toDataURL(latestQR, { width: 400 });
    res.send(`<h2>Scan with WhatsApp</h2><img src="${qrImage}" />`);
});

const PORT = 5000; 
app.listen(PORT, () => console.log(`🌐 Web monitor on port ${PORT}`));

// --------------------
// CONFIG & SECRETS
// --------------------
const FOLDER_ID = process.env.DRIVE_FOLDER_ID;
const OWNER = (process.env.OWNER_NUMBER || '923000000000') + '@s.whatsapp.net';

const driveAuth = new google.auth.GoogleAuth({
    credentials: JSON.parse(process.env.GOOGLE_DRIVE_KEY || '{}'),
    scopes: ['https://www.googleapis.com/auth/drive.readonly']
});
const drive = google.drive({ version: 'v3', auth: driveAuth });

const genAI = new GoogleGenerativeAI(process.env.GEMINI_API_KEY || '');
const model = genAI.getGenerativeModel({ model: 'gemini-2.5-flash' });

// --------------------
// BOT ENGINE
// --------------------
async function startBot() {
    const { state, saveCreds } = await useMultiFileAuthState('auth_info');
    const { version } = await fetchLatestBaileysVersion();

    const sock = makeWASocket({
        version,
        auth: state,
        logger: pino({ level: 'silent' }),
        browser: Browsers.macOS('Desktop'),
    });

    sock.ev.on('creds.update', saveCreds);

    sock.ev.on('connection.update', async (update) => {
        const { connection, lastDisconnect, qr } = update;

        if (qr) {
            latestQR = qr;
            console.log('✨ SCAN THE QR CODE BELOW:');
            qrcodeTerminal.generate(qr, { small: true }); 
        }

        if (connection === 'open') {
            latestQR = null;
            console.log('✅ WhatsApp Connected!');
            await sock.sendMessage(OWNER, { text: '🏥 *System Online*' });
        }

        if (connection === 'close') {
            const shouldReconnect = lastDisconnect?.error?.output?.statusCode !== DisconnectReason.loggedOut;
            if (shouldReconnect) startBot();
        }
    });

    sock.ev.on('messages.upsert', async ({ messages }) => {
        const msg = messages[0];
        if (!msg.message || msg.key.fromMe) return;

        const jid = msg.key.remoteJid;
        const body = msg.message.conversation || msg.message.extendedTextMessage?.text || '';
        const text = body.toLowerCase().trim();

        if (text === '.menu') {
            return await sock.sendMessage(jid, { text: '🏥 *Commands:*\n• .report [Date]\n• .ai [Question]' });
        }

        if (text.startsWith('report')) {
            const date = text.split(' ')[1];
            if (!date) return await sock.sendMessage(jid, { text: '❌ Usage: report 14012026' });

            try {
                const res = await drive.files.list({
                    q: `'${FOLDER_ID}' in parents and name = '${date}.png'`,
                    fields: 'files(id, name)'
                });

                if (res.data.files.length > 0) {
                    const imgRes = await drive.files.get({ fileId: res.data.files[0].id, alt: 'media' }, { responseType: 'arraybuffer' });
                    const mainImg = await Jimp.read(Buffer.from(imgRes.data));

                    if (fs.existsSync('./logo.png')) {
                        const logo = await Jimp.read('./logo.png');
                        logo.resize({ w: 150 });
                        mainImg.composite(logo, mainImg.width - 170, mainImg.height - 100);
                    }

                    const buffer = await mainImg.getBuffer('image/png');
                    await sock.sendMessage(jid, { image: buffer, caption: `✅ Report for ${date}` });
                } else {
                    await sock.sendMessage(jid, { text: '❌ Report not found.' });
                }
            } catch (e) { console.error(e); }
        }

        if (text.startsWith('.ai')) {
            try {
                const result = await model.generateContent(body.slice(4));
                await sock.sendMessage(jid, { text: `🤖 AI: ${result.response.text()}` });
            } catch (e) { console.error(e); }
        }
    });
}

startBot();
