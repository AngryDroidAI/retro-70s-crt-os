<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<title>Retro 70s CRT OS</title>
<style>
  html, body {margin:0; padding:0; overflow:hidden; background:black;}
  canvas {display:block;}
  #overlay {position:absolute; top:0; left:0; color:#0f0; font:14px 'Courier New', monospace; pointer-events:none;}
  #metrics {position:absolute; top:10px; right:10px; color:#0f0; font:12px 'Courier New', monospace; background:rgba(0,0,0,0.5); padding:5px; 
border:1px solid #0f0; display:none;}
</style>
</head>
<body>
<canvas id="crt"></canvas>
<div id="metrics"></div>
<script>
/* -----------------  CONFIGURATION  ----------------- */
const CHAR_W = 8;      // character width in pixels
const CHAR_H = 16;     // character height in pixels
const COLS   = 80;     // columns per row
const ROWS   = 25;     // rows per screen
const FPS    = 60;     // animation frames per second
const SCANLINE_ALPHA = 32; // transparency of scanlines (0‑255)
const FLICKER_LEVEL  = 0.05; // noise strength (0‑1)

/* -----------------  STATE  ----------------- */
const fsKey = 'crt_os_fs';   // key in localStorage
let files = {};              // file → content
let directories = {};         // directory → {files: [], subdirs: []}
let currentDir = '/';         // current working directory
let screenBuffer = [];       // array of strings (one per row)
let cursor = {x:0, y:0};
let inputBuffer = '';
let scrollOffset = 0;        // for scrolling older lines
let commandHistory = [];
let historyIndex = -1;
let currentTheme = 'green';  // 'green', 'amber', 'blue'
let isEditing = false;       // track if we're in edit mode
let editFile = '';           // file being edited
let editContent = [];        // content lines for editing
let editCursor = {x:0, y:0}; // cursor position in editor
let showMetrics = false;     // track whether to show system metrics

// System metrics (simulated)
let cpuUsage = 0;
let memoryUsage = 0;
let storageUsed = 0;
let systemUptime = 0;

/* -----------------  THEMES  ----------------- */
const themes = {
  green: { text: '#0f0', scanline: 'rgba(0,0,0,0.2)' },
  amber: { text: '#ffa500', scanline: 'rgba(0,0,0,0.15)' },
  blue: { text: '#00ffff', scanline: 'rgba(0,0,0,0.25)' }
};

/* -----------------  INITIALISATION  ----------------- */
const canvas = document.getElementById('crt');
const metricsDiv = document.getElementById('metrics');
const ctx = canvas.getContext('2d');
ctx.font = `${CHAR_H}px 'Courier New', monospace`;
ctx.textBaseline = 'top';
ctx.fillStyle = themes[currentTheme].text;

function resize() {
  canvas.width = window.innerWidth;
  canvas.height = window.innerHeight;
}
window.addEventListener('resize', resize);
resize();

// Initialize system uptime timer
setInterval(() => {
  systemUptime++;
}, 1000);

// Initialize metrics simulation
function updateMetrics() {
  // Simulate CPU usage fluctuations
  cpuUsage = 5 + Math.sin(systemUptime/10) * 20 + Math.random() * 15;
  cpuUsage = Math.min(Math.max(cpuUsage, 5), 85);
  
  // Calculate memory usage
  const fileCount = Object.keys(files).length;
  const dirCount = Object.keys(directories).length;
  const totalSize = JSON.stringify({files, directories}).length;
  storageUsed = totalSize;
  memoryUsage = Math.min(75 + (fileCount + dirCount) % 25, 95);
}

// load file system
function loadFS() {
  const data = localStorage.getItem(fsKey);
  if (data) {
    const parsed = JSON.parse(data);
    files = parsed.files || {};
    directories = parsed.directories || {};
    currentTheme = parsed.theme || 'green';
  }
}
function saveFS() {
  localStorage.setItem(fsKey, JSON.stringify({
    files, 
    directories, 
    theme: currentTheme
  }));
}
loadFS();

// initialise screen buffer with empty lines
for (let r=0;r<ROWS;r++) screenBuffer.push('');

bootScreen();

/* -----------------  RENDERING  ----------------- */
function render() {
  // Update system metrics
  updateMetrics();
  
  // 1 Clear background
  ctx.fillStyle = '#000';
  ctx.fillRect(0,0,canvas.width,canvas.height);

  // 2 Draw text with current theme
  ctx.fillStyle = themes[currentTheme].text;
  for (let r=0;r<ROWS;r++) {
    const line = screenBuffer[(r+scrollOffset)%ROWS];
    ctx.fillText(line, 0, r*CHAR_H);
  }

  // 3 Draw prompt line with user input
  const prompt = `${currentDir}$ ${inputBuffer}`;
  ctx.fillText(prompt, 0, ROWS*CHAR_H);

  // 4 Scan-lines
  const scan = ctx.createLinearGradient(0,0,0,canvas.height);
  scan.addColorStop(0,'rgba(0,0,0,0)');
  scan.addColorStop(0.5, themes[currentTheme].scanline);
  scan.addColorStop(1,'rgba(0,0,0,0)');
  ctx.fillStyle = scan;
  ctx.fillRect(0,0,canvas.width,canvas.height);

  // 5 Flicker (simple per-pixel noise)
  const imgData = ctx.getImageData(0,0,canvas.width,canvas.height);
  const data = imgData.data;
  for (let i=0;i<data.length;i+=4) {
    const d = Math.random()*255*FLICKER_LEVEL;
    data[i] = data[i]   - d; // R
    data[i+1] = data[i+1]- d; // G
    data[i+2] = data[i+2]- d; // B
  }
  ctx.putImageData(imgData,0,0);
  
  // Render metrics panel if enabled
  if (showMetrics) {
    renderMetrics();
  }
}

/* Render system metrics panel */
function renderMetrics() {
  metricsDiv.style.color = themes[currentTheme].text;
  metricsDiv.style.borderColor = themes[currentTheme].text;
  metricsDiv.style.display = 'block';
  
  const fileCount = Object.keys(files).length;
  const dirCount = Object.keys(directories).length;
  
  // Format uptime
  const hours = Math.floor(systemUptime / 3600);
  const minutes = Math.floor((systemUptime % 3600) / 60);
  const seconds = systemUptime % 60;
  const uptimeStr = `${hours.toString().padStart(2, '0')}:${minutes.toString().padStart(2, '0')}:${seconds.toString().padStart(2, '0')}`;
  
  metricsDiv.innerHTML = `
    <div>CPU: ${cpuUsage.toFixed(1)}%</div>
    <div>RAM: ${memoryUsage.toFixed(1)}%</div>
    <div>Storage: ${storageUsed} bytes</div>
    <div>Files: ${fileCount}</div>
    <div>Dirs: ${dirCount}</div>
    <div>Uptime: ${uptimeStr}</div>
  `;
}

/* -----------------  INPUT HANDLING  ----------------- */
window.addEventListener('keydown', e=>{
  if (e.ctrlKey || e.metaKey) return;

  // Toggle metrics with F1
  if (e.key === 'F1') {
    showMetrics = !showMetrics;
    if (!showMetrics) {
      metricsDiv.style.display = 'none';
    }
    return;
  }

  if (isEditing) {
    handleEditMode(e);
    return;
  }

  if (e.key === 'Enter') {
    if (inputBuffer.trim()) {
      commandHistory.push(inputBuffer);
      historyIndex = commandHistory.length;
    }
    processCommand(inputBuffer.trim());
    inputBuffer = '';
  } else if (e.key === 'Backspace') {
    inputBuffer = inputBuffer.slice(0,-1);
  } else if (e.key === 'ArrowUp') {
    if (historyIndex > 0) {
      historyIndex--;
      inputBuffer = commandHistory[historyIndex];
    }
  } else if (e.key === 'ArrowDown') {
    if (historyIndex < commandHistory.length - 1) {
      historyIndex++;
      inputBuffer = commandHistory[historyIndex];
    } else {
      historyIndex = commandHistory.length;
      inputBuffer = '';
    }
  } else if (e.key === 'Tab') {
    e.preventDefault();
    autoComplete(inputBuffer);
  } else if (e.key.length === 1 && e.key.match(/[\x20-\x7e]/)) {
    inputBuffer += e.key;
  } else if (e.key === 'ArrowLeft') {
    scrollOffset = (scrollOffset + 1) % ROWS;
  } else if (e.key === 'ArrowRight') {
    scrollOffset = (scrollOffset - 1 + ROWS) % ROWS;
  }
});

function handleEditMode(e) {
  if (e.key === 'Escape') {
    exitEditMode();
  } else if (e.key === 'Enter') {
    // Insert new line
    const currentLine = editContent[editCursor.y] || '';
    const beforeCursor = currentLine.substring(0, editCursor.x);
    const afterCursor = currentLine.substring(editCursor.x);
    
    editContent[editCursor.y] = beforeCursor;
    editContent.splice(editCursor.y + 1, 0, afterCursor);
    
    editCursor.y++;
    editCursor.x = 0;
    
    redrawEditScreen();
  } else if (e.key === 'Backspace') {
    if (editCursor.x > 0) {
      // Delete character before cursor
      const line = editContent[editCursor.y];
      editContent[editCursor.y] = line.substring(0, editCursor.x - 1) + line.substring(editCursor.x);
      editCursor.x--;
    } else if (editCursor.y > 0) {
      // Merge with previous line
      const currentLine = editContent[editCursor.y] || '';
      const prevLine = editContent[editCursor.y - 1] || '';
      editContent[editCursor.y - 1] = prevLine + currentLine;
      editContent.splice(editCursor.y, 1);
      editCursor.y--;
      editCursor.x = prevLine.length;
    }
    redrawEditScreen();
  } else if (e.key === 'Delete') {
    const line = editContent[editCursor.y] || '';
    if (editCursor.x < line.length) {
      // Delete character at cursor
      editContent[editCursor.y] = line.substring(0, editCursor.x) + line.substring(editCursor.x + 1);
    } else if (editCursor.y < editContent.length - 1) {
      // Merge with next line
      const nextLine = editContent[editCursor.y + 1] || '';
      editContent[editCursor.y] = line + nextLine;
      editContent.splice(editCursor.y + 1, 1);
    }
    redrawEditScreen();
  } else if (e.key === 'ArrowLeft') {
    if (editCursor.x > 0) {
      editCursor.x--;
    } else if (editCursor.y > 0) {
      editCursor.y--;
      editCursor.x = (editContent[editCursor.y] || '').length;
    }
  } else if (e.key === 'ArrowRight') {
    const line = editContent[editCursor.y] || '';
    if (editCursor.x < line.length) {
      editCursor.x++;
    } else if (editCursor.y < editContent.length - 1) {
      editCursor.y++;
      editCursor.x = 0;
    }
  } else if (e.key === 'ArrowUp') {
    if (editCursor.y > 0) {
      editCursor.y--;
      const lineLength = (editContent[editCursor.y] || '').length;
      if (editCursor.x > lineLength) {
        editCursor.x = lineLength;
      }
    }
  } else if (e.key === 'ArrowDown') {
    if (editCursor.y < editContent.length - 1) {
      editCursor.y++;
      const lineLength = (editContent[editCursor.y] || '').length;
      if (editCursor.x > lineLength) {
        editCursor.x = lineLength;
      }
    }
  } else if (e.key === 'Home') {
    editCursor.x = 0;
  } else if (e.key === 'End') {
    const line = editContent[editCursor.y] || '';
    editCursor.x = line.length;
  } else if (e.key.length === 1 && e.key.match(/[\x20-\x7e]/)) {
    // Insert character
    if (editContent.length === 0) editContent.push('');
    const line = editContent[editCursor.y] || '';
    editContent[editCursor.y] = line.substring(0, editCursor.x) + e.key + line.substring(editCursor.x);
    editCursor.x++;
    redrawEditScreen();
  }
}

function redrawEditScreen() {
  clearScreen();
  writeLine(`Editing: ${editFile} (Press ESC to save and exit)`);
  writeLine('='.repeat(COLS));
  editContent.forEach((line, index) => {
    if (index === editCursor.y) {
      // Show cursor position
      const cursorLine = line.substring(0, editCursor.x) + '|' + line.substring(editCursor.x);
      writeLine(cursorLine);
    } else {
      writeLine(line);
    }
  });
}

function exitEditMode() {
  files[editFile] = editContent.join('\n');
  saveFS();
  isEditing = false;
  editFile = '';
  editContent = [];
  editCursor = {x:0, y:0};
  writeLine(`File ${editFile} saved.`);
}

/* -----------------  COMMAND PROCESSING  ----------------- */
function processCommand(cmd) {
  writeLine(`${currentDir}$ ${cmd}`);
  if (!cmd) return;

  const parts = cmd.split(/\s+/);
  const cmdName = parts[0];
  const args = parts.slice(1);

  switch(cmdName) {
    case 'help':    help();           break;
    case 'echo':    echo(args);       break;
    case 'ls':      ls(args[0]);      break;
    case 'cat':     cat(args[0]);     break;
    case 'clear':   clearScreen();    break;
    case 'mkdir':   mkdir(args[0]);   break;
    case 'rm':      rm(args[0]);      break;
    case 'save':    saveFS(); writeLine('FS saved'); break;
    case 'load':    loadFS(); writeLine('FS loaded'); break;
    case 'pwd':     pwd();            break;
    case 'cd':      cd(args[0]);      break;
    case 'theme':   changeTheme(args[0]); break;
    case 'edit':    edit(args[0]);    break;
    case 'date':    showDate();       break;
    case 'time':    showTime();       break;
    case 'calc':    calculate(args);  break;
    case 'find':    find(args[0]);    break;
    case 'history': showHistory();    break;
    case 'reboot':  reboot();         break;
    case 'mem':     showMemory();     break;
    case 'sysinfo': showSysInfo();    break;
    case 'metrics': toggleMetrics(args[0]); break;
    default:        // Try to save as file
      const contentIndex = cmd.indexOf(' ');
      if (contentIndex > 0) {
        const fname = cmd.substring(0, contentIndex);
        const content = cmd.substring(contentIndex + 1);
        files[fname] = content;
        saveFS();
        writeLine(`Saved ${fname}`);
      } else {
        writeLine(`Unknown command: ${cmdName}`);
      }
      break;
  }
}

function writeLine(text) {
  screenBuffer[cursor.y] = text.substring(0, COLS).padEnd(COLS, ' ');
  cursor.y = (cursor.y + 1) % ROWS;
  scrollOffset = 0;
}

/* -----------------  ENHANCED SHELL COMMANDS  ----------------- */
function help() {
  writeLine('Available commands:');
  writeLine('  help           - show this help');
  writeLine('  echo text      - print text');
  writeLine('  ls [dir]       - list files/dirs');
  writeLine('  cat file       - show file contents');
  writeLine('  clear          - clear screen');
  writeLine('  mkdir dir      - create directory');
  writeLine('  rm file        - delete file/dir');
  writeLine('  pwd            - show current directory');
  writeLine('  cd dir         - change directory');
  writeLine('  theme [color]  - change theme (green/amber/blue)');
  writeLine('  edit file      - edit file content');
  writeLine('  date           - show current date');
  writeLine('  time           - show current time');
  writeLine('  calc expr      - simple calculator');
  writeLine('  find pattern   - search files');
  writeLine('  history        - show command history');
  writeLine('  reboot         - restart system');
  writeLine('  save/load      - save/load file system');
  writeLine('  mem            - show memory usage');
  writeLine('  sysinfo        - show system information');
  writeLine('  <file> <content> - quick save to file');
  writeLine('  metrics [on|off] - toggle system metrics');
  writeLine('  F1             - toggle metrics panel');
}

function ls(dir = '.') {
  const targetDir = dir === '.' ? currentDir : resolvePath(dir);
  const names = Object.keys(files).filter(f => f.startsWith(targetDir))
                .concat(Object.keys(directories).filter(d => d.startsWith(targetDir)))
                .sort();
  if (names.length === 0) writeLine('No files or directories.');
  else names.forEach(n => {
    const name = n.replace(targetDir, '').replace(/^\//, '');
    if (name) writeLine(name);
  });
}

function cat(file) {
  if (!file) { writeLine('Usage: cat <file>'); return; }
  const fullPath = resolvePath(file);
  if (files.hasOwnProperty(fullPath)) {
    const content = files[fullPath];
    content.split('\n').forEach(line => writeLine(line));
  } else {
    writeLine(`cat: ${file}: No such file`);
  }
}

function clearScreen() {
  for (let r=0;r<ROWS;r++) screenBuffer[r]='';
  cursor.y = 0;
  scrollOffset = 0;
}

function mkdir(dir) {
  if (!dir) { writeLine('Usage: mkdir <dir>'); return; }
  const fullPath = resolvePath(dir);
  if (directories.hasOwnProperty(fullPath)) { 
    writeLine(`mkdir: ${dir} already exists`); 
    return; 
  }
  directories[fullPath] = { files: [], subdirs: [] };
  saveFS();
  writeLine(`Directory ${dir} created`);
}

function rm(name) {
  if (!name) { writeLine('Usage: rm <file|dir>'); return; }
  const fullPath = resolvePath(name);
  if (files.hasOwnProperty(fullPath)) {
    delete files[fullPath];
    saveFS();
    writeLine(`Removed file ${name}`);
  } else if (directories.hasOwnProperty(fullPath)) {
    delete directories[fullPath];
    saveFS();
    writeLine(`Removed directory ${name}`);
  } else {
    writeLine(`rm: ${name}: No such file or directory`);
  }
}

function pwd() {
  writeLine(currentDir);
}

function cd(dir) {
  if (!dir) { 
    currentDir = '/'; 
    writeLine('Returned to root directory');
    return; 
  }
  const newDir = resolvePath(dir);
  if (directories.hasOwnProperty(newDir)) {
    currentDir = newDir;
    writeLine(`Changed to ${currentDir}`);
  } else {
    writeLine(`cd: ${dir}: No such directory`);
  }
}

function changeTheme(color) {
  if (!color) {
    writeLine(`Current theme: ${currentTheme}`);
    writeLine('Available: green, amber, blue');
    return;
  }
  if (themes[color]) {
    currentTheme = color;
    ctx.fillStyle = themes[currentTheme].text;
    saveFS();
    writeLine(`Theme changed to ${color}`);
  } else {
    writeLine(`Unknown theme: ${color}`);
  }
}

function edit(file) {
  if (!file) { writeLine('Usage: edit <file>'); return; }
  const fullPath = resolvePath(file);
  editFile = fullPath;
  editContent = files.hasOwnProperty(fullPath) ? files[fullPath].split('\n') : [''];
  editCursor = {x:0, y:0};
  isEditing = true;
  redrawEditScreen();
}

function showDate() {
  writeLine(new Date().toDateString());
}

function showTime() {
  writeLine(new Date().toLocaleTimeString());
}

function calculate(args) {
  if (args.length === 0) {
    writeLine('Usage: calc <expression>');
    return;
  }
  try {
    const expr = args.join(' ');
    // Safe evaluation
    const result = Function(`"use strict"; return (${expr})`)();
    writeLine(`${expr} = ${result}`);
  } catch (e) {
    writeLine('Calculation error');
  }
}

function find(pattern) {
  if (!pattern) {
    writeLine('Usage: find <pattern>');
    return;
  }
  const matches = Object.keys(files).filter(f => f.includes(pattern));
  if (matches.length === 0) {
    writeLine('No matches found');
  } else {
    matches.forEach(m => writeLine(m));
  }
}

function showHistory() {
  if (commandHistory.length === 0) {
    writeLine('No command history');
    return;
  }
  commandHistory.forEach((cmd, i) => writeLine(`${i + 1}: ${cmd}`));
}

function reboot() {
  clearScreen();
  bootScreen();
}

function showMemory() {
  const fileCount = Object.keys(files).length;
  const dirCount = Object.keys(directories).length;
  const totalSize = JSON.stringify({files, directories}).length;
  writeLine(`Files: ${fileCount}`);
  writeLine(`Directories: ${dirCount}`);
  writeLine(`Storage used: ${totalSize} bytes`);
}

function showSysInfo() {
  writeLine('Retro 70s CRT OS v2.1');
  writeLine('Enhanced Edition');
  writeLine('');
  writeLine('System Information:');
  writeLine(`  Theme: ${currentTheme}`);
  writeLine(`  Resolution: ${COLS}x${ROWS}`);
  writeLine(`  Character size: ${CHAR_W}x${CHAR_H}`);
  writeLine(`  Storage: localStorage`);
  writeLine(`  FPS: ${FPS}`);
}

function toggleMetrics(arg) {
  if (!arg) {
    showMetrics = !showMetrics;
    writeLine(`Metrics panel ${showMetrics ? 'enabled' : 'disabled'}`);
    if (!showMetrics) {
      metricsDiv.style.display = 'none';
    }
  } else if (arg === 'on') {
    showMetrics = true;
    writeLine('Metrics panel enabled');
  } else if (arg === 'off') {
    showMetrics = false;
    metricsDiv.style.display = 'none';
    writeLine('Metrics panel disabled');
  } else {
    writeLine('Usage: metrics [on|off]');
  }
}

function autoComplete(input) {
  const parts = input.split(' ');
  const lastPart = parts[parts.length - 1];
  const commands = ['help', 'echo', 'ls', 'cat', 'clear', 'mkdir', 'rm', 'pwd', 'cd', 'theme', 'edit', 'date', 'time', 'calc', 'find', 
'history', 'reboot', 'mem', 'sysinfo', 'metrics'];
  const matches = commands.filter(cmd => cmd.startsWith(lastPart));
  
  if (matches.length === 1) {
    parts[parts.length - 1] = matches[0];
    inputBuffer = parts.join(' ') + ' ';
  } else if (matches.length > 1) {
    writeLine('Possible completions:');
    matches.forEach(m => writeLine(`  ${m}`));
  }
}

function resolvePath(path) {
  if (path.startsWith('/')) return path;
  if (currentDir === '/') return `/${path}`;
  return `${currentDir}/${path}`;
}

/* -----------------  BOOT SCREEN  ----------------- */
function bootScreen() {
  const splash = [
    '  ____   ____   ___   ____    ___   _____  ___  ',
    ' / __ \\ / __ \\ / _ \\ / __ \\  / _ \\ |  __ \\/ __| ',
    '| |  | | |  | | | | | |  | || | | || |__) | (__  ',
    '| |  | | |  | | | | | |  | || | | ||  _  / \\___| ',
    '| |__| | |__| | |_| | |__| || |_| || | \\ \\  ___  ',
    ' \\____/ \\____/ \\___/ \\____/  \\___/ |_|  \\_\\_|   ',
    '',
    '  Retro 70s CRT OS (v2.1) - Enhanced Edition',
    '  All data kept in localStorage',
    '',
    '  Type "help" to see available commands.',
    '',
    `  Current theme: ${currentTheme}`,
    `  Current directory: ${currentDir}`,
    '',
    '  System ready. Press any key to start...',
  ];
  
  for (let r=0;r<ROWS;r++) {
    screenBuffer[r] = splash[r] ? splash[r].padEnd(COLS, ' ') : '';
  }
  
  cursor.y = Math.min(splash.length, ROWS);
  scrollOffset = 0;
}

/* -----------------  MAIN LOOP  ----------------- */
function main() {
  render();
  setTimeout(main, 1000/FPS);
}
main();
</script>
</body>
</html>
