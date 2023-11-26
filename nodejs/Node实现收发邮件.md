发邮件用 `SMTP` 协议
收邮件用 `POP3/IMAP` 协议
## 发送邮件
从Node.js发送电子邮件-简单如蛋糕！🍰✉️
[nodemailer](https://github.com/nodemailer/nodemailer)
比如QQ邮箱，在设置中开启SMTP协议服务
根据GPT提供的代码填入对应的options
```js
const nodemailer = require('nodemailer')

// 创建传输器对象
const transporter = nodemailer.createTransport({
  service: 'qq', // 选择邮件服务提供商，例如Gmail、QQ邮箱等
  auth: {
    user: '1445709137@qq.com', // 发件人邮箱
    pass: 'gbtixhhoyfjugcgi', // 发件人邮箱密码或授权码
  },
})

// 邮件选项
const mailOptions = {
  from: '1445709137@qq.com', // 发件人邮箱
  to: '1445709137@qq.com', // 收件人邮箱
  subject: 'Hello from Node.js', // 邮件主题
  // 支持html+css
  text: 'Hello, this is a test email from Node.js!', // 邮件内容
}

// 发送邮件
transporter.sendMail(mailOptions, (error, info) => {
  if (error) {
    console.log('Error occurred:', error.message)
  } else {
    console.log('Email sent successfully!')
    console.log('Message ID:', info.messageId)
  }
})
```
![[Pasted image 20231126190253.png]]
## 获取邮件
通过 [imap](https://www.npmjs.com/package/imap) 实现了邮件的搜索，然后用 [mailparser](mailparser)来做了内容解析，然后把邮件内容和附件做了下载
```js
const { MailParser } = require('mailparser') // 引入MailParser库
const fs = require('fs') // 引入文件系统模块
const path = require('path') // 引入路径模块
const Imap = require('imap') // 引入Imap库

var imap = new Imap({
  user: '1445709137@qq.com', // 邮箱用户名
  password: 'gbtixhhoyfjugcgi', // 邮箱密码或授权码
  host: 'imap.qq.com', // IMAP服务器地址
  port: 993, // IMAP服务器端口
  tls: true, // 使用TLS加密连接
})

// 连接成功事件
imap.once('ready', () => {
  imap.openBox('INBOX', true, (err) => {
    // 打开收件箱
    imap.search([['SEEN'], ['SINCE', new Date('2023-11-26').toLocaleString()]], (err, results) => {
      if (!err) {
        handleResults(results) // 处理搜索结果
      } else {
        throw err
      }
    })
  })
})

function handleResults(results) {
  imap
    .fetch(results, {
      bodies: '',
    })
    .on('message', (msg) => {
      const mailparser = new MailParser() // 创建MailParser实例

      msg.on('body', (stream) => {
        const info = {} // 存储邮件信息的对象
        stream.pipe(mailparser) // 将邮件内容流传递给MailParser

        mailparser.on('headers', (headers) => {
          // 解析邮件头部信息
          info.theme = headers.get('subject') // 主题
          info.from = headers.get('from').value[0].address // 发件人邮箱地址
          info.mailName = headers.get('from').value[0].name // 发件人姓名
          info.to = headers.get('to').value[0].address // 收件人邮箱地址
          info.datatime = headers.get('date').toLocaleString() // 发送日期时间
        })

        mailparser.on('data', (data) => {
          if (data.type === 'text') {
            // 邮件内容为文本类型
            info.html = data.html // HTML内容
            info.text = data.text // 纯文本内容

            const filePath = path.join(__dirname, 'mails', info.theme + '.html')
            fs.writeFileSync(filePath, info.html || info.text) // 将邮件内容写入文件

            console.log(info) // 打印邮件信息
          }
          if (data.type === 'attachment') {
            // 邮件内容为附件类型
            const filePath = path.join(__dirname, 'files', data.filename)
            const ws = fs.createWriteStream(filePath)
            data.content.pipe(ws) // 将附件内容写入文件
          }
        })
      })
    })
}

imap.connect() // 连接到邮件服务器
```