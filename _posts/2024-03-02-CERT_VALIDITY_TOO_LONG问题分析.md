# CERT_VALIDITY_TOO_LONG问题分析

## 先说结论：

>  在 2012 年 7 月 1 日 BR 生效日期当天或之后颁发的证书：60几个月。
>  最后可能的到期日：2015 年 4 月 1 日 + 60 个月 = 2020-04-01
>
>   2015 年 4 月 1 日或之后颁发的证书：39 个月。
>   最后可能到期日：2018 年 3 月 1 日 + 39 个月 = 2021-06-01
>
>   2018 年 3 月 1 日或之后颁发的证书：825 天。
>   最后可能到期时间：2020 年 9 月 1 日 + 825 天 = 2022-12-05
>
>   当前限制，来自 Chrome 根证书策略：
>   2020 年 9 月 1 日或之后颁发的证书：398 天。

## 分析

WebView打印如下日志：

```txt
21:53:24.993  1979-2020  chromium  [ERROR:ssl_client_socket_impl.cc(995)] handshake failed; returned -1, SSL error code 1, net_error -213
```
`Java`层异常为：

```java
Caused by: javax.net.ssl.SSLProtocolException: Read error: ssl=0x3fff07a6f808: Failure in SSL library, usually a protocol error
error:10000416:SSL routines:OPENSSL_internal:SSLV3_ALERT_CERTIFICATE_UNKNOWN(external/boringssl/src/ssl/tls_record.cc:592 0x3fff07a7f1c8:0x00000001)
at com.android.org.conscrypt.NativeCrypto.ENGINE_SSL_read_direct(Native Method)
at com.android.org.conscrypt.NativeSsl.readDirectByteBuffer(NativeSsl.java:521)
at com.android.org.conscrypt.ConscryptEngine.readPlaintextDataDirect(ConscryptEngine.java:1099)
at com.android.org.conscrypt.ConscryptEngine.readPlaintextDataHeap(ConscryptEngine.java:1119)
at com.android.org.conscrypt.ConscryptEngine.readPlaintextData(ConscryptEngine.java:1091)
at com.android.org.conscrypt.ConscryptEngine.unwrap(ConscryptEngine.java:841)
... 55 more
```

根据这2条报错信息，在中可以找到如下代码片段：
```cpp
// net/socket/ssl_client_socket_impl.cc

int SSLClientSocketImpl::DoHandshake() {
  crypto::OpenSSLErrStackTracer err_tracer(FROM_HERE);

  int rv = SSL_do_handshake(ssl_.get());
  int net_error = OK;
  if (rv <= 0) {
    int ssl_error = SSL_get_error(ssl_.get(), rv);
    if (ssl_error == SSL_ERROR_WANT_X509_LOOKUP && !send_client_cert_) {
      return ERR_SSL_CLIENT_AUTH_CERT_NEEDED;
    }
    if (ssl_error == SSL_ERROR_WANT_PRIVATE_KEY_OPERATION) {
      DCHECK(client_private_key_);
      DCHECK_NE(kSSLClientSocketNoPendingResult, signature_result_);
      next_handshake_state_ = STATE_HANDSHAKE;
      return ERR_IO_PENDING;
    }
    if (ssl_error == SSL_ERROR_WANT_CERTIFICATE_VERIFY) {
      DCHECK(cert_verifier_request_);
      next_handshake_state_ = STATE_HANDSHAKE;
      return ERR_IO_PENDING;
    }

    OpenSSLErrorInfo error_info;
    net_error = MapLastOpenSSLError(ssl_error, err_tracer, &error_info);
    if (net_error == ERR_IO_PENDING) {
      // If not done, stay in this state
      next_handshake_state_ = STATE_HANDSHAKE;
      return ERR_IO_PENDING;
    }
	//日志打印处
    LOG(ERROR) << "handshake failed; returned " << rv << ", SSL error code "
               << ssl_error << ", net_error " << net_error;
    NetLogOpenSSLError(net_log_, NetLogEventType::SSL_HANDSHAKE_ERROR,
                       net_error, ssl_error, error_info);
  }
```

搜索错误代码可以找到如下错误定义：

```cpp
// net/base/net_error_list.h

// The certificate's validity period is too long.
NET_ERROR(CERT_VALIDITY_TOO_LONG, -213)
```

再搜索`CERT_VALIDITY_TOO_LONG`，定位到

```cpp
// net/cert/cert_status_flags.cc
// 将证书验证状态映射为具体错误代码
int MapCertStatusToNetError(CertStatus cert_status) {
  // A certificate may have multiple errors.  We report the most
  // serious error.

  // Unrecoverable errors
  if (cert_status & CERT_STATUS_INVALID)
    return ERR_CERT_INVALID;
  if (cert_status & CERT_STATUS_PINNED_KEY_MISSING)
    return ERR_SSL_PINNED_KEY_NOT_IN_CERT_CHAIN;

  // Potentially recoverable errors
  if (cert_status & CERT_STATUS_KNOWN_INTERCEPTION_BLOCKED)
    return ERR_CERT_KNOWN_INTERCEPTION_BLOCKED;
  if (cert_status & CERT_STATUS_REVOKED)
    return ERR_CERT_REVOKED;
  if (cert_status & CERT_STATUS_AUTHORITY_INVALID)
    return ERR_CERT_AUTHORITY_INVALID;
  if (cert_status & CERT_STATUS_COMMON_NAME_INVALID)
    return ERR_CERT_COMMON_NAME_INVALID;
  if (cert_status & CERT_STATUS_CERTIFICATE_TRANSPARENCY_REQUIRED)
    return ERR_CERTIFICATE_TRANSPARENCY_REQUIRED;
  if (cert_status & CERT_STATUS_SYMANTEC_LEGACY)
    return ERR_CERT_SYMANTEC_LEGACY;
  // CERT_STATUS_NON_UNIQUE_NAME is intentionally not mapped to an error.
  // It is treated as just a warning and used to degrade the SSL UI.
  if (cert_status & CERT_STATUS_NAME_CONSTRAINT_VIOLATION)
    return ERR_CERT_NAME_CONSTRAINT_VIOLATION;
  if (cert_status & CERT_STATUS_WEAK_SIGNATURE_ALGORITHM)
    return ERR_CERT_WEAK_SIGNATURE_ALGORITHM;
  if (cert_status & CERT_STATUS_WEAK_KEY)
    return ERR_CERT_WEAK_KEY;
  if (cert_status & CERT_STATUS_DATE_INVALID)
    return ERR_CERT_DATE_INVALID;
  // 定位到此处
  if (cert_status & CERT_STATUS_VALIDITY_TOO_LONG)
    return ERR_CERT_VALIDITY_TOO_LONG;
  if (cert_status & CERT_STATUS_UNABLE_TO_CHECK_REVOCATION)
    return ERR_CERT_UNABLE_TO_CHECK_REVOCATION;
  if (cert_status & CERT_STATUS_NO_REVOCATION_MECHANISM)
    return ERR_CERT_NO_REVOCATION_MECHANISM;

  // Unknown status. The assumption is 0 (an OK status) won't be used here.
  NOTREACHED();
  return ERR_UNEXPECTED;
}

// net/cert/cert_verify_proc.cc
// 证书验证函数中，调用了MapCertStatusToNetError()函数
int CertVerifyProc::Verify(X509Certificate* cert,
                           const std::string& hostname,
                           const std::string& ocsp_response,
                           const std::string& sct_list,
                           int flags,
                           CertVerifyResult* verify_result,
                           const NetLogWithSource& net_log,
                           std::optional<base::Time> time_now) {
  CHECK(cert);
  CHECK(verify_result);

  net_log.BeginEvent(NetLogEventType::CERT_VERIFY_PROC, [&] {
    return CertVerifyParams(cert, hostname, ocsp_response, sct_list, flags,
                            crl_set());
  });
  // CertVerifyProc's contract allows ::VerifyInternal() to wait on File I/O
  // (such as the Windows registry or smart cards on all platforms) or may re-
  // enter this code via extension hooks (such as smart card UI). To ensure
  // threads are not starved or deadlocked, the base::ScopedBlockingCall below
  // increments the thread pool capacity when this method takes too much time to
  // run.
  base::ScopedBlockingCall scoped_blocking_call(FROM_HERE,
                                                base::BlockingType::MAY_BLOCK);

  verify_result->Reset();
  verify_result->verified_cert = cert;

  int rv = VerifyInternal(cert, hostname, ocsp_response, sct_list, flags,
                          verify_result, net_log, time_now);

  CHECK(verify_result->verified_cert);

  // Check for mismatched signature algorithms and unknown signature algorithms
  // in the chain. Also fills in the has_* booleans for the digest algorithms
  // present in the chain.
  // 验证签名算法
  if (!InspectSignatureAlgorithmsInChain(verify_result)) {
    verify_result->cert_status |= CERT_STATUS_INVALID;
    rv = MapCertStatusToNetError(verify_result->cert_status);
  }
  // 验证域名与SAN扩展中的域名或者ip是否匹配
  if (!cert->VerifyNameMatch(hostname)) {
    verify_result->cert_status |= CERT_STATUS_COMMON_NAME_INVALID;
    rv = MapCertStatusToNetError(verify_result->cert_status);
  }

  if (verify_result->ocsp_result.response_status ==
      bssl::OCSPVerifyResult::NOT_CHECKED) {
    // If VerifyInternal did not record the result of checking stapled OCSP,
    // do it now.
    BestEffortCheckOCSP(ocsp_response, *verify_result->verified_cert,
                        &verify_result->ocsp_result);
  }

  // Check to see if the connection is being intercepted.
  for (const auto& hash : verify_result->public_key_hashes) {
    if (hash.tag() != HASH_VALUE_SHA256) {
      continue;
    }
    if (!crl_set()->IsKnownInterceptionKey(std::string_view(
            reinterpret_cast<const char*>(hash.data()), hash.size()))) {
      continue;
    }

    if (verify_result->cert_status & CERT_STATUS_REVOKED) {
      // If the chain was revoked, and a known MITM was present, signal that
      // with a more meaningful error message.
      verify_result->cert_status |= CERT_STATUS_KNOWN_INTERCEPTION_BLOCKED;
      rv = MapCertStatusToNetError(verify_result->cert_status);
    } else {
      // Otherwise, simply signal informatively. Both statuses are not set
      // simultaneously.
      verify_result->cert_status |= CERT_STATUS_KNOWN_INTERCEPTION_DETECTED;
    }
    break;
  }

  // 验证SAN
  std::vector<std::string> dns_names, ip_addrs;
  cert->GetSubjectAltName(&dns_names, &ip_addrs);
  if (HasNameConstraintsViolation(verify_result->public_key_hashes,
                                  cert->subject().common_name,
                                  dns_names,
                                  ip_addrs)) {
    verify_result->cert_status |= CERT_STATUS_NAME_CONSTRAINT_VIOLATION;
    rv = MapCertStatusToNetError(verify_result->cert_status);
  }

  // Check for weak keys in the entire verified chain.
  // 验证弱密钥
  bool weak_key = ExaminePublicKeys(verify_result->verified_cert,
                                    verify_result->is_issued_by_known_root);

  if (weak_key) {
    verify_result->cert_status |= CERT_STATUS_WEAK_KEY;
    // Avoid replacing a more serious error, such as an OS/library failure,
    // by ensuring that if verification failed, it failed with a certificate
    // error.
    if (rv == OK || IsCertificateError(rv))
      rv = MapCertStatusToNetError(verify_result->cert_status);
  }

  if (verify_result->has_sha1)
    verify_result->cert_status |= CERT_STATUS_SHA1_SIGNATURE_PRESENT;

  // Flag certificates using weak signature algorithms.
  // sha1已过时
  bool sha1_allowed = (flags & VERIFY_ENABLE_SHA1_LOCAL_ANCHORS) &&
                      !verify_result->is_issued_by_known_root;
  if (!sha1_allowed && verify_result->has_sha1) {
    verify_result->cert_status |= CERT_STATUS_WEAK_SIGNATURE_ALGORITHM;
    // Avoid replacing a more serious error, such as an OS/library failure,
    // by ensuring that if verification failed, it failed with a certificate
    // error.
    if (rv == OK || IsCertificateError(rv))
      rv = MapCertStatusToNetError(verify_result->cert_status);
  }

  // Distrust Symantec-issued certificates, as described at
  // https://security.googleblog.com/2017/09/chromes-plan-to-distrust-symantec.html
  // 旧版Symantec证书
  if (!(flags & VERIFY_DISABLE_SYMANTEC_ENFORCEMENT) &&
      IsLegacySymantecCert(verify_result->public_key_hashes)) {
    verify_result->cert_status |= CERT_STATUS_SYMANTEC_LEGACY;
    if (rv == OK || IsCertificateError(rv))
      rv = MapCertStatusToNetError(verify_result->cert_status);
  }

  // Flag certificates from publicly-trusted CAs that are issued to intranet
  // hosts. While the CA/Browser Forum Baseline Requirements (v1.1) permit
  // these to be issued until 1 November 2015, they represent a real risk for
  // the deployment of gTLDs and are being phased out ahead of the hard
  // deadline.
  if (verify_result->is_issued_by_known_root && IsHostnameNonUnique(hostname)) {
    verify_result->cert_status |= CERT_STATUS_NON_UNIQUE_NAME;
    // CERT_STATUS_NON_UNIQUE_NAME will eventually become a hard error. For
    // now treat it as a warning and do not map it to an error return value.
  }

  // Flag certificates using too long validity periods.
  // 验证证书有效期
  if (verify_result->is_issued_by_known_root && HasTooLongValidity(*cert)) {
    verify_result->cert_status |= CERT_STATUS_VALIDITY_TOO_LONG;
    if (rv == OK)
      rv = MapCertStatusToNetError(verify_result->cert_status);
  }

  // Record a histogram for per-verification usage of root certs.
  if (rv == OK) {
    RecordTrustAnchorHistogram(verify_result->public_key_hashes,
                               verify_result->is_issued_by_known_root);
  }

  net_log.EndEvent(NetLogEventType::CERT_VERIFY_PROC,
                   [&] { return verify_result->NetLogParams(rv); });
  return rv;
}

int CertVerifyProcAndroid::VerifyInternal(X509Certificate* cert,
                                          const std::string& hostname,
                                          const std::string& ocsp_response,
                                          const std::string& sct_list,
                                          int flags,
                                          CertVerifyResult* verify_result,
                                          const NetLogWithSource& net_log,
                                          std::optional<base::Time>) {
  std::vector<std::string> cert_bytes;
  GetChainDEREncodedBytes(cert, &cert_bytes);
  // 验证Android TrustManager
  if (!VerifyFromAndroidTrustManager(cert_bytes, hostname, flags,
                                     cert_net_fetcher_, verify_result)) {
    return ERR_FAILED;
  }

  if (IsCertStatusError(verify_result->cert_status))
    return MapCertStatusToNetError(verify_result->cert_status);

  if (TestRootCerts::HasInstance() &&
      !verify_result->verified_cert->intermediate_buffers().empty() &&
      TestRootCerts::GetInstance()->IsKnownRoot(x509_util::CryptoBufferAsSpan(
          verify_result->verified_cert->intermediate_buffers().back().get()))) {
    verify_result->is_issued_by_known_root = true;
  }

  LogNameNormalizationMetrics(".Android", verify_result->verified_cert.get(),
                              verify_result->is_issued_by_known_root);

  return OK;
}

```

我们定位到验证有效期函数是`HasTooLongValidity(*cert)`, 

```cpp
// static
bool CertVerifyProc::HasTooLongValidity(const X509Certificate& cert) {
  const base::Time& start = cert.valid_start();
  const base::Time& expiry = cert.valid_expiry();
  if (start.is_max() || start.is_null() || expiry.is_max() ||
      expiry.is_null() || start > expiry) {
    return true;
  }

  // The maximum lifetime of publicly trusted certificates has reduced
  // gradually over time. These dates are derived from the transitions noted in
  // Section 1.2.2 (Relevant Dates) of the Baseline Requirements.
  //
  // * Certificates issued before BRs took effect, Chrome limited to max of ten
  // years validity and a max notAfter date of 2019-07-01.
  //   * Last possible expiry: 2019-07-01.
  //
  // * Cerificates issued on-or-after the BR effective date of 1 July 2012: 60
  // months.
  //   * Last possible expiry: 1 April 2015 + 60 months = 2020-04-01
  //
  // * Certificates issued on-or-after 1 April 2015: 39 months.
  //   * Last possible expiry: 1 March 2018 + 39 months = 2021-06-01
  //
  // * Certificates issued on-or-after 1 March 2018: 825 days.
  //   * Last possible expiry: 1 September 2020 + 825 days = 2022-12-05
  //
  // The current limit, from Chrome Root Certificate Policy:
  // * Certificates issued on-or-after 1 September 2020: 398 days.

  // 公共信任证书的最长生命周期已缩短
  // 随着时间的推移逐渐。这些日期来自于中记录的转变
  // 基准要求第 1.2.2 节（相关日期）。
  //
  // * 在 BR 生效之前颁发的证书，Chrome 最多只能有 10 个
  // 有效期为 2019 年 7 月 1 日。
  // * 最后可能到期时间：2019-07-01。
  //
  // * 在 2012 年 7 月 1 日 BR 生效日期当天或之后颁发的证书：60
  // 几个月。
  // * 最后可能的到期日：2015 年 4 月 1 日 + 60 个月 = 2020-04-01
  //
  // * 2015 年 4 月 1 日或之后颁发的证书：39 个月。
  // * 最后可能到期日：2018 年 3 月 1 日 + 39 个月 = 2021-06-01
  //
  // * 2018 年 3 月 1 日或之后颁发的证书：825 天。
  // * 最后可能到期时间：2020 年 9 月 1 日 + 825 天 = 2022-12-05
  //
  // 当前限制，来自 Chrome 根证书策略：
  // * 2020 年 9 月 1 日或之后颁发的证书：398 天。
  base::TimeDelta validity_duration = cert.valid_expiry() - cert.valid_start();

  // No certificates issued before the latest lifetime requirement was enacted
  // could possibly still be accepted, so we don't need to check the older
  // limits explicitly.
  return validity_duration > base::Days(398);
}
```



