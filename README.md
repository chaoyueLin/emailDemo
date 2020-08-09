# k9
[k9mail](https://github.com/k9mail/k-9)是andorid的邮件app.本文是使用学习的一些记录
邮件协议分Header,Body两部分

	Date: Tue, 30 Jan 2018 19:45:40 +0000 (GMT)
	From: Apple <noreply@email.apple.com>
	To: 364065814@qq.com
	Message-ID: <384093730.21631892.1517341540713.JavaMail.email@email.apple.com>
	Subject: =?UTF-8?B?6Ie05Lit5Zu95YaF5ZywIGlDbG91ZCDnlKjmiLfnmoTph43opoHpgJrnn6U=?=

Body邮件正文有part部分，有简单的文字text/plain,有html,text/plain.当有多个或是有附件是需要有

	multipart/lternative

	Mime-Version: 1.0
	Content-Type: multipart/alternative;
		boundary="----=_NextPart_5A72B828_0980A998_20E304BA"
	Content-Transfer-Encoding: 8Bit
	This is a multi-part message in MIME format.

	------=_NextPart_5A72B828_0980A998_20E304BA
	Content-Type: text/plain;
		charset="gb18030"
	Content-Transfer-Encoding: base64

	vPu8+8Tj

	------=_NextPart_5A72B828_0980A998_20E304BA
	Content-Type: text/html;
	charset="gb18030"
	Content-Transfer-Encoding: base64

	PGRpdj68+7z7xOM8L2Rpdj48ZGl2PiA8L2Rpdj4=

	------=_NextPart_5A72B828_0980A998_20E304BA--
	
Part是邮件内容协议的接口定义，header设置协议中Content-Type,Content-Transfer-Encoding等，邮件内容通过流读取，写入，就是Body

	public interface Part extends IPartID{
    void addHeader(String name, String value);

    void addRawHeader(String name, String raw);

    void removeHeader(String name);

    void setHeader(String name, String value);

    Body getBody();

    String getContentType();

    String getDisposition();

    String getContentId();

    /**
     * Returns an array of headers of the given name. The array may be empty.
     */
    @NonNull
    String[] getHeader(String name);

    boolean isMimeType(String mimeType);

    String getMimeType();

    void setBody(Body body);

    void writeTo(OutputStream out) throws IOException, MessagingException;

    void writeHeaderTo(OutputStream out) throws IOException, MessagingException;

    String getServerExtra();

    void setServerExtra(String serverExtra);
	}
	public interface Body {
    /**
     * Returns the raw data of the body, without transfer encoding etc applied.
     * TODO perhaps it would be better to have an intermediate "simple part" class 	where this method could reside
     *    because it makes no sense for multiparts
     */
    public InputStream getInputStream() throws MessagingException;

    /**
     * Sets the content transfer encoding (7bit, 8bit, quoted-printable or base64).
     */
    public void setEncoding(String encoding) throws MessagingException;

    /**
     * Writes the body's data to the given {@link OutputStream}.
     * The written data is transfer encoded (e.g. transformed to Base64 when 	needed).
     */
    public void writeTo(OutputStream out) throws IOException, MessagingException;
	}
	

邮件内容body内容文本，文件作为附件上传实现在MessageBuilder#builderBody，里面有组合多个part方法，添加附件part方法。MessageBuilder#builderHeader是添加Header信息。

	private void buildBody(MimeMessage message) throws MessagingException {
    // Build the body.
    // TODO FIXME - body can be either an HTML or Text part, depending on whether 	we're in
    // HTML mode or not.  Should probably fix this so we don't mix up html and text 	parts.
    TextBody body = buildText(isDraft);

    // text/plain part when messageFormat == MessageFormat.HTML
    TextBody bodyPlain = null;

    final boolean hasAttachments = !attachments.isEmpty();

    if (messageFormat == SimpleMessageFormat.HTML) {
        // HTML message (with alternative text part)

        // This is the compiled MIME part for an HTML message.
        MimeMultipart composedMimeMessage = createMimeMultipart();
        composedMimeMessage.setSubType("alternative");
        // Let the receiver select either the text or the HTML part.
        bodyPlain = buildText(isDraft, SimpleMessageFormat.TEXT);
        composedMimeMessage.addBodyPart(new MimeBodyPart(bodyPlain, "text/plain"));
        composedMimeMessage.addBodyPart(new MimeBodyPart(body, "text/html"));

        if (hasAttachments) {
            // If we're HTML and have attachments, we have a MimeMultipart 	container to hold the
            // whole message (mp here), of which one part is a MimeMultipart 	container
            // (composedMimeMessage) with the user's composed messages, and 	subsequent parts for
            // the attachments.
            MimeMultipart mp = createMimeMultipart();
            mp.addBodyPart(new MimeBodyPart(composedMimeMessage));
            addAttachmentsToMessage(mp);
            MimeMessageHelper.setBody(message, mp);
        } else {
            // If no attachments, our multipart/alternative part is the only one we 	need.
            MimeMessageHelper.setBody(message, composedMimeMessage);
        }
    } else if (messageFormat == SimpleMessageFormat.TEXT) {
        // Text-only message.
        if (hasAttachments) {
            MimeMultipart mp = createMimeMultipart();
            mp.addBodyPart(new MimeBodyPart(body, "text/plain"));
            addAttachmentsToMessage(mp);
            MimeMessageHelper.setBody(message, mp);
        } else {
            // No attachments to include, just stick the text body in the message 	and call it good.
            MimeMessageHelper.setBody(message, body);
        }
    }

    // If this is a draft, add metadata for thawing.
    if (isDraft) {
        // Add the identity to the message.
        message.addHeader(K9.IDENTITY_HEADER, buildIdentityHeader(body, 	bodyPlain));
    }
	}
	private void buildHeader(MimeMessage message) throws MessagingException {
    message.addSentDate(sentDate, hideTimeZone);
    Address from = new Address(identity.getEmail(), identity.getName());
    message.setFrom(from);
    message.setRecipients(RecipientType.TO, to);
    message.setRecipients(RecipientType.CC, cc);
    message.setRecipients(RecipientType.BCC, bcc);
    message.setSubject(subject);

    if (requestReadReceipt) {
        message.setHeader("Disposition-Notification-To", from.toEncodedString());
        message.setHeader("X-Confirm-Reading-To", from.toEncodedString());
        message.setHeader("Return-Receipt-To", from.toEncodedString());
    }

    if (!K9.hideUserAgent()) {
        message.setHeader("User-Agent", OkHttpConfig.getUserAgent(true));
    }

    final String replyTo = identity.getReplyTo();
    if (replyTo != null) {
        message.setReplyTo(new Address[] { new Address(replyTo) });
    }

    if (inReplyTo != null) {
        message.setInReplyTo(inReplyTo);
    }

    if (references != null) {
        message.setReferences(references);
    }

    String messageId = messageIdGenerator.generateMessageId(message);
    message.setMessageId(messageId);

    if (isDraft && isPgpInlineEnabled) {
        message.setFlag(Flag.X_DRAFT_OPENPGP_INLINE, true);
    }

    //添加统计需要的头
    message.setHeader(Consts.HEADER_FM_CLIENTTYPE, Consts.HEADER_CLIENTTYPE_ANDROID 	+ AppUtils.getAppVersionName());
	}
	
邮件正文有附件，或是有需要多个part时需要最外层有multipart/alternative，Multipart用双亲树形结构。
	public abstract class Multipart implements Body {
    private Part mParent;

    private final List<BodyPart> mParts = new ArrayList<BodyPart>();

    public void addBodyPart(BodyPart part) {
        mParts.add(part);
        part.setParent(this);
    }

    public BodyPart getBodyPart(int index) {
        return mParts.get(index);
    }

    public List<BodyPart> getBodyParts() {
        return Collections.unmodifiableList(mParts);
    }

    public abstract String getMimeType();

    public abstract String getBoundary();

    public int getCount() {
        return mParts.size();
    }
    
邮件内容文本，附件都是通过流上传，下载，文本用TextBody,附件用TempFileBody.附件比较简单是文件的字节读取，文本比较复杂，需要对换行，html标签的处理等，所以有TextBodyBuilder类。有两个方法buildTextHtml,buildTextPlain，对应内容的text/html,text/plain.

	public TextBody buildTextHtml() {
    // The length of the formatted version of the user-supplied text/reply
    int composedMessageLength;

    // The offset of the user-supplied text/reply in the final text body
    int composedMessageOffset;

    // Get the user-supplied text
    String text = mMessageContent;

    // Do we have to modify an existing message to include our reply?
    if (mIncludeQuotedText) {
        InsertableHtmlContent quotedHtmlContent = getQuotedTextHtml();

        if (K9.isDebug()) {
            Timber.d("insertable: %s", quotedHtmlContent.toDebugString());
        }

        if (mAppendSignature) {
            // Append signature to the reply
            if (mReplyAfterQuote || mSignatureBeforeQuotedText) {
                text += getSignature();
            }
        }

        // Convert the text to HTML
        text = textToHtmlFragment(text);

        /*
         * Set the insertion location based upon our reply after quote
         * setting. Additionally, add some extra separators between the
         * composed message and quoted message depending on the quote
         * location. We only add the extra separators when we're
         * sending, that way when we load a draft, we don't have to know
         * the length of the separators to remove them before editing.
         */
        if (mReplyAfterQuote) {
            quotedHtmlContent.setInsertionLocation(
                    InsertableHtmlContent.InsertionLocation.AFTER_QUOTE);
            if (mInsertSeparator) {
                text = "<br clear=\"all\">" + text;
            }
        } else {
            quotedHtmlContent.setInsertionLocation(
                    InsertableHtmlContent.InsertionLocation.BEFORE_QUOTE);
            if (mInsertSeparator) {
                text += "<br><br>";
            }
        }

        if (mAppendSignature) {
            // Place signature immediately after the quoted text
            if (!(mReplyAfterQuote || mSignatureBeforeQuotedText)) {
                quotedHtmlContent.insertIntoQuotedFooter(getSignatureHtml());
            }
        }

        quotedHtmlContent.setUserContent(text);

        // Save length of the body and its offset.  This is used when thawing 	drafts.
        composedMessageLength = text.length();
        composedMessageOffset = quotedHtmlContent.getInsertionPoint();
        text = quotedHtmlContent.toString();

    } else {
        // There is no text to quote so simply append the signature if available
        if (mAppendSignature) {
            text += getSignature();
        }

        // Convert the text to HTML
        text = textToHtmlFragment(text);

        text= HtmlQuoteCreator.warpHtml(text);

        composedMessageLength = text.length();
        composedMessageOffset = 0;
    }

    TextBody body = new TextBody(text);
    body.setComposedMessageLength(composedMessageLength);
    body.setComposedMessageOffset(composedMessageOffset);

    return body;
	}



