# SDP-TO-JSON

interface CodecPayload {
  channels: number;
  clockrate: number;
  id: string;
  name: string;
  parameters?: any;
}

/**
 * WebrtcCodec: static variables describe each supported codecs for detection
 */
class WebrtcCodec {
  static readonly VP8 = new WebrtcCodec('video', {
    channels: 1,
    clockrate: 90000,
    id: '96',
    name: 'VP8',
  });

  static readonly VP9 = new WebrtcCodec('video', {
    channels: 1,
    clockrate: 90000,
    id: '100',
    name: 'VP9',
  });

  static readonly H264 = new WebrtcCodec('video', {
    channels: 1,
    clockrate: 90000,
    id: '97',
    name: 'H264',
    parameters: [
      { key: 'profile-level-id', value: '42C01F' },
      { key: 'packetization-mode', value: '1' },
    ],
  });

  static readonly OPUS = new WebrtcCodec('audio', {
    channels: 2,
    clockrate: 48000,
    id: '111',
    name: 'OPUS',
  });

  static readonly ISAC = new WebrtcCodec('audio', {
    channels: 1,
    clockrate: 16000,
    id: '104',
    name: 'ISAC',
  });

  private constructor(public readonly type: 'audio' | 'video', public readonly payload: CodecPayload) {}
}

/**
 * Checks if the specified codec is supported for receiving webrtc video or audio
 *
 * Sets up a pseudo server offer, with only the specified codec, and checks if local answer covers that codec
 *
 * @param codec one of WebrtcCodec static variables
 * @return true if codec is supported
 */
async function isWebrtcReceiveCodecSupported(codec: WebrtcCodec): Promise<boolean> {
  try {
    // build offer from server with specified codec (minimal and incomplete)
    const sdpOfferDesc = {
      groups: [{ contents: [codec.type], semantics: 'BUNDLE' }],
      contents: [
        {
          name: codec.type,
          application: {
            applicationType: 'rtp',
            media: codec.type,
            mux: true,
            payloads: [codec.payload],
          },
          transport: {
            candidates: [],
            fingerprints: [{ hash: 'sha-256', setup: 'actpass', value: Array(32).fill('00').join(':') }],
            pwd: '0'.repeat(22),
            transportType: 'iceUdp',
            ufrag: '0'.repeat(8),
          },
        },
      ],
    };
    const sdpOffer = toSessionSDP(sdpOfferDesc, { role: 'initiator', direction: 'outgoing' });

    // set up peerconnection to negotiate SDPs
    const peerConnection = new RTCPeerConnection({ iceServers: [] });
    await peerConnection.setRemoteDescription(new RTCSessionDescription({ type: 'offer', sdp: sdpOffer }));
    const sdpAnswer = await peerConnection.createAnswer();
    const sdpAnswerDesc = toSessionJSON(sdpAnswer.sdp, { role: 'responder', direction: 'outgoing' });

    // process answer SDP
    if (sdpAnswerDesc.contents === undefined) return false;
    for (const content of sdpAnswerDesc.contents) {
      if (content.application?.payloads === undefined) continue;
      for (const payload of content.application.payloads) {
        // check if answer payload has the same codec name - other parameters are not checked
        if (payload?.name?.toUpperCase() === codec.payload.name.toUpperCase()) return true;
      }
    }

    // if none of the contents/payloads matches the requested codec, codec is not supported
    return false;
  } catch (e) {
    // Safari throws exception when a codec is not supported - this is not an error
    return false;
  }
}

/**
 * Checks if the specified codec is supported for publishing webrtc video or audio
 *
 * This must called with a previously acquired MediaStream object.
 * Make sure that the stream has the appropriate channels (audio and/or video) for the codec decetion
 *
 * @param mediaStream a previously acquire MediaStream object
 * @param codec one of WebrtcCodec static variables
 * @return true if codec is supported
 */
async function isWebrtcPublishCodecSupported(mediaStream: MediaStream, codec: WebrtcCodec): Promise<boolean> {
  try {
    // setup peerconnection
    const peerConnection = new RTCPeerConnection({ iceServers: [] });
    if (codec.type === 'video') {
      for (const track of mediaStream.getVideoTracks()) {
        peerConnection.addTrack(track, mediaStream);
      }
    } else {
      // 'audio'
      for (const track of mediaStream.getAudioTracks()) {
        peerConnection.addTrack(track, mediaStream);
      }
    }

    const sdpOffer = await peerConnection.createOffer();
    const sdpOfferDesc = toSessionJSON(sdpOffer.sdp, { role: 'initiator', direction: 'outgoing' });

    // process offer SDP
    if (sdpOfferDesc.contents === undefined) return false;
    for (const content of sdpOfferDesc.contents) {
      if (content.application?.payloads === undefined) continue;
      for (const payload of content.application.payloads) {
        // check if answer payload has the same codec name - other parameters are not checked
        if (payload?.name?.toUpperCase() === codec.payload.name.toUpperCase()) return true;
      }
    }

    return false;
  } catch (e) {
    // Safari throws exception when a codec is not supported - this is not an error
    return false;
  }
}


const toSessionSDP = function (session, opts) {
    var role = opts.role || 'initiator';
    var direction = opts.direction || 'outgoing';
    var sid = opts.sid || session.sid || Date.now();
    var time = opts.time || Date.now();

    var sdp = [
        'v=0',
        'o=- ' + sid + ' ' + time + ' IN IP4 0.0.0.0',
        's=-',
        't=0 0'
    ];

    var contents = session.contents || [];
    var hasSources = false;
    contents.forEach(function (content) {
        if (content.application.sources &&
            content.application.sources.length) {
            hasSources = true;
        }
    });

    if (hasSources) {
        sdp.push('a=msid-semantic: WMS *');
    }

    var groups = session.groups || [];
    groups.forEach(function (group) {
        sdp.push('a=group:' + group.semantics + ' ' + group.contents.join(' '));
    });


    contents.forEach(function (content) {
        sdp.push(toMediaSDP(content, opts));
    });

    return sdp.join('\r\n') + '\r\n';
};

const toMediaSDP = function (content, opts) {
    var sdp = [];

    var role = opts.role || 'initiator';
    var direction = opts.direction || 'outgoing';

    var desc = content.application;
    var transport = content.transport;
    var payloads = desc.payloads || [];
    var fingerprints = (transport && transport.fingerprints) || [];

    var mline = [];
    if (desc.applicationType == 'datachannel') {
        mline.push('application');
        mline.push('1');
        mline.push('DTLS/SCTP');
        if (transport.sctp) {
            transport.sctp.forEach(function (map) {
                mline.push(map.number);
            });
        }
    } else {
        mline.push(desc.media);
        mline.push('1');
        if (fingerprints.length > 0) {
            mline.push('UDP/TLS/RTP/SAVPF');
        } else if (desc.encryption && desc.encryption.length > 0) {
            mline.push('RTP/SAVPF');
        } else {
            mline.push('RTP/AVPF');
        }
        payloads.forEach(function (payload) {
            mline.push(payload.id);
        });
    }


    sdp.push('m=' + mline.join(' '));

    sdp.push('c=IN IP4 0.0.0.0');
    if (desc.bandwidth && desc.bandwidth.type && desc.bandwidth.bandwidth) {
        sdp.push('b=' + desc.bandwidth.type + ':' + desc.bandwidth.bandwidth);
    }
    if (desc.applicationType == 'rtp') {
        sdp.push('a=rtcp:1 IN IP4 0.0.0.0');
    }

    if (transport) {
        if (transport.ufrag) {
            sdp.push('a=ice-ufrag:' + transport.ufrag);
        }
        if (transport.pwd) {
            sdp.push('a=ice-pwd:' + transport.pwd);
        }

        var pushedSetup = false;
        fingerprints.forEach(function (fingerprint) {
            sdp.push('a=fingerprint:' + fingerprint.hash + ' ' + fingerprint.value);
            if (fingerprint.setup && !pushedSetup) {
                sdp.push('a=setup:' + fingerprint.setup);
            }
        });

        if (transport.sctp) {
            transport.sctp.forEach(function (map) {
                sdp.push('a=sctpmap:' + map.number + ' ' + map.protocol + ' ' + map.streams);
            });
        }
    }

    if (desc.applicationType == 'rtp') {
        sdp.push('a=' + (SENDERS[role][direction][content.senders] || 'sendrecv'));
    }
    sdp.push('a=mid:' + content.name);

    if (desc.sources && desc.sources.length) {
        var streams = {};
        desc.sources.forEach(function (source) {
            (source.parameters || []).forEach(function (param) {
                if (param.key === 'msid') {
                    streams[param.value] = 1;
                }
            });
        });
        streams = Object.keys(streams);
        if (streams.length === 1) {
            sdp.push('a=msid:' + streams[0]);
        }
    }

    if (desc.mux) {
        sdp.push('a=rtcp-mux');
    }
    if (desc.rsize) {
        sdp.push('a=rtcp-rsize');
    }

    var encryption = desc.encryption || [];
    encryption.forEach(function (crypto) {
        sdp.push('a=crypto:' + crypto.tag + ' ' + crypto.cipherSuite + ' ' + crypto.keyParams + (crypto.sessionParams ? ' ' + crypto.sessionParams : ''));
    });
    if (desc.googConferenceFlag) {
        sdp.push('a=x-google-flag:conference');
    }

    payloads.forEach(function (payload) {
        var rtpmap = 'a=rtpmap:' + payload.id + ' ' + payload.name + '/' + payload.clockrate;
        if (payload.channels && payload.channels != '1') {
            rtpmap += '/' + payload.channels;
        }
        sdp.push(rtpmap);

        if (payload.parameters && payload.parameters.length) {
            var fmtp = ['a=fmtp:' + payload.id];
            var parameters = [];
            payload.parameters.forEach(function (param) {
                parameters.push((param.key ? param.key + '=' : '') + param.value);
            });
            fmtp.push(parameters.join(';'));
            sdp.push(fmtp.join(' '));
        }

        if (payload.feedback) {
            payload.feedback.forEach(function (fb) {
                if (fb.type === 'trr-int') {
                    sdp.push('a=rtcp-fb:' + payload.id + ' trr-int ' + (fb.value ? fb.value : '0'));
                } else {
                    sdp.push('a=rtcp-fb:' + payload.id + ' ' + fb.type + (fb.subtype ? ' ' + fb.subtype : ''));
                }
            });
        }
    });

    if (desc.feedback) {
        desc.feedback.forEach(function (fb) {
            if (fb.type === 'trr-int') {
                sdp.push('a=rtcp-fb:* trr-int ' + (fb.value ? fb.value : '0'));
            } else {
                sdp.push('a=rtcp-fb:* ' + fb.type + (fb.subtype ? ' ' + fb.subtype : ''));
            }
        });
    }

    var hdrExts = desc.headerExtensions || [];
    hdrExts.forEach(function (hdr) {
        sdp.push('a=extmap:' + hdr.id + (hdr.senders ? '/' + SENDERS[role][direction][hdr.senders] : '') + ' ' + hdr.uri);
    });

    var ssrcGroups = desc.sourceGroups || [];
    ssrcGroups.forEach(function (ssrcGroup) {
        sdp.push('a=ssrc-group:' + ssrcGroup.semantics + ' ' + ssrcGroup.sources.join(' '));
    });

    var ssrcs = desc.sources || [];
    ssrcs.forEach(function (ssrc) {
        for (var i = 0; i < ssrc.parameters.length; i++) {
            var param = ssrc.parameters[i];
            sdp.push('a=ssrc:' + (ssrc.ssrc || desc.ssrc) + ' ' + param.key + (param.value ? (':' + param.value) : ''));
        }
    });

    var candidates = transport.candidates || [];
    candidates.forEach(function (candidate) {
        sdp.push(toCandidateSDP(candidate));
    });

    return sdp.join('\r\n');
};

const toCandidateSDP = function (candidate) {
    var sdp = [];

    sdp.push(candidate.foundation);
    sdp.push(candidate.component);
    sdp.push(candidate.protocol.toUpperCase());
    sdp.push(candidate.priority);
    sdp.push(candidate.ip);
    sdp.push(candidate.port);

    var type = candidate.type;
    sdp.push('typ');
    sdp.push(type);
    if (type === 'srflx' || type === 'prflx' || type === 'relay') {
        if (candidate.relAddr && candidate.relPort) {
            sdp.push('raddr');
            sdp.push(candidate.relAddr);
            sdp.push('rport');
            sdp.push(candidate.relPort);
        }
    }
    if (candidate.tcpType && candidate.protocol.toUpperCase() == 'TCP') {
        sdp.push('tcptype');
        sdp.push(candidate.tcpType);
    }

    sdp.push('generation');
    sdp.push(candidate.generation || '0');

    // FIXME: apparently this is wrong per spec
    // but then, we need this when actually putting this into
    // SDP so it's going to stay.
    // decision needs to be revisited when browsers dont
    // accept this any longer
    return 'a=candidate:' + sdp.join(' ');
};


const SENDERS = {
    initiator: {
        incoming: {
            initiator: 'recvonly',
            responder: 'sendonly',
            both: 'sendrecv',
            none: 'inactive',
            recvonly: 'initiator',
            sendonly: 'responder',
            sendrecv: 'both',
            inactive: 'none'
        },
        outgoing: {
            initiator: 'sendonly',
            responder: 'recvonly',
            both: 'sendrecv',
            none: 'inactive',
            recvonly: 'responder',
            sendonly: 'initiator',
            sendrecv: 'both',
            inactive: 'none'
        }
    },
    responder: {
        incoming: {
            initiator: 'sendonly',
            responder: 'recvonly',
            both: 'sendrecv',
            none: 'inactive',
            recvonly: 'responder',
            sendonly: 'initiator',
            sendrecv: 'both',
            inactive: 'none'
        },
        outgoing: {
            initiator: 'recvonly',
            responder: 'sendonly',
            both: 'sendrecv',
            none: 'inactive',
            recvonly: 'initiator',
            sendonly: 'responder',
            sendrecv: 'both',
            inactive: 'none'
        }
    }
};

isWebrtcReceiveCodecSupported(WebrtcCodec.H264).then((supported)=>{
  console.log("supported", supported);
});
