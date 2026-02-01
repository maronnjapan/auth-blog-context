---
title: "Node Crypto、Web Crypto APIを用いてECDSAによる秘密鍵・公開鍵の作成～署名・検証まで行う"
emoji: "🙆"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["ECDSA","Crypto","Node"]
published: true
---
## はじめに
JavaScriptには鍵周りを取り扱うことができるCryptoがあります。
このCryptoを使用することで、共有鍵や公開鍵・秘密鍵を作成できます。
なので今回はCryptoを使用して、以下のことをやっていきます。
- 秘密鍵・公開鍵の作成
- 共有鍵による秘密鍵の暗号化
- 秘密鍵による署名と公開鍵による署名の検証

ただ、Cryptoといっても[Web Crypto API](https://developer.mozilla.org/ja/docs/Web/API/Crypto)と[Node.js Crypto](https://nodejs.org/api/crypto.html)が存在しています。
そこで、それぞれを単体で使用した時と鍵の作成はWeb Crypto APIを使用して、署名・検証部分はNode.js Cryptoの形式に変換するコードを展開します。
なお、各種暗号アルゴリズムやCryptoは私自身習熟していません。
上記の解説部分には誤りを含む可能性があるため、正しい知識は別途信頼できる書籍や文書などで補完いただけますと幸いです。
実装する流れを参考にしていただきたい気持ちでこの記事を書いております。
## 記事を読み進める前の注意点
ブログを書いている人はセキュリティ分野のスペシャリストではありません。
そのため、今回の鍵を作成する手法が安全であることは保証できません。
[Web Crypto APIのドキュメント](https://developer.mozilla.org/ja/docs/Web/API/Web_Crypto_API)の最初にも以下のような記載がある通り、Web Crypto APIを使って、singからverifyができたといってそれが本当に安全な方法かはわかりません。
![image.png](/images/node-and-web-crypto-generate-ecdsa-key/image.png)
なので、基本的には実績のあるライブラリなどを使用するか、セキュリティのスペシャリストにお墨付きをもらって運用を行う必要があります。
## コード全体
コード全体を示します。
### NodeのCryptoのみの実装
ESDSAの公開鍵・秘密鍵の作成と秘密鍵の暗号用の共有鍵を作成しています。
```tsx
import { v4 as uuidv4 } from 'uuid'
import * as crypto from 'node:crypto';
import { Injectable } from '@nestjs/common';
import { PrismaClient } from '@/__generated__';
const KEY_BASE_OPTIONS = {
    namedCurve: "P-256",
} as const;
@Injectable()
export class SignKeyOnlyNodeCryptoService {
    private prisma: PrismaClient
    constructor() {
        this.prisma = new PrismaClient();
    }
    async generateSignKeys() {
        /**
         * ECDSAの公開鍵・秘密鍵のペアを生成
         */
        const keyPair = crypto.generateKeyPairSync('ec', {
            ...KEY_BASE_OPTIONS,
            publicKeyEncoding: {
                type: 'spki',
                format: 'pem'
            },
            privateKeyEncoding: {
                type: 'pkcs8',
                format: 'pem'
            }
        });
        /** 公開鍵をJWK形式に変換 */
        const publicKeyObj = crypto.createPublicKey(keyPair.publicKey);
        const publicKeyJwk = publicKeyObj.export({ format: 'jwk' });
        return {
            kid: uuidv4(),
            publicKey: publicKeyJwk,
            privateKey: keyPair.privateKey,
        }
    }
    async saveKeys(kid: string, keys: { publicKey: any, privateKey: string }): Promise<void> {
        /**
         * 秘密鍵を暗号化するためのAESキーを取得
         */
        let wrapKey = (await this.prisma.wrapKey.findFirst())?.key
        if (!wrapKey) {
            /**
             * 秘密鍵を暗号化するためのAESキーが存在しない場合は作成
             */
            wrapKey = await this.findOrCreateWrapKey();
        }
        /**
         * 公開鍵はすでにJWK形式で受け取っている
         * JWTで使用するためにuse, algプロパティを追加
         */
        const publicKeyJwk = {
            ...keys.publicKey,
            use: 'sig',
            alg: 'ES256'
        };
        /**
         * 初期化ベクトルを生成
         */
        const iv = crypto.randomBytes(12);
        /**
         * 秘密鍵をAES-GCMのアルゴリズムで暗号化
         */
        const privateKeyPem = keys.privateKey;
        /**
         * 秘密鍵を暗号化するためのAESキーをインポート
         */
        const keyBuffer = Buffer.from(wrapKey, 'base64');
        const aesKey = crypto.createSecretKey(keyBuffer);
        /**
         * 暗号化
         */
        const cipher = crypto.createCipheriv('aes-256-gcm', aesKey, iv);
        const encryptedPrivateKey = Buffer.concat([
            cipher.update(Buffer.from(privateKeyPem, 'utf-8')),
            cipher.final()
        ]);
        /**
         * 認証タグを取得
         * Node.jsのCryptoでは、別途認証タグを取得する必要がある
         */
        const authTag = cipher.getAuthTag();
        /**
         * 暗号化データ、認証タグを結合して保存
         */
        const wrappedPrivateKey = Buffer.concat([encryptedPrivateKey, authTag]);
        /**
         * DBへの保存
         */
        try {
            await this.prisma.privateKey.create({
                data: {
                    kid,
                    /**
                     * IVと暗号化された秘密鍵をBase64で保存
                     */
                    privateKey: `${iv.toString('base64')}:${wrappedPrivateKey.toString('base64')}`,
                }
            })
            await this.prisma.publicKey.create({
                data: {
                    kid,
                    publicKey: JSON.stringify(publicKeyJwk),
                }
            })
        } catch (e) {
            await this.prisma.privateKey.delete({
                where: {
                    kid
                }
            })
            await this.prisma.publicKey.delete({
                where: {
                    kid
                }
            })
            throw e;
        }
    }
    /** AESの鍵を作成し、DBに保存 */
    async findOrCreateWrapKey() {
        const wrapKey = await this.prisma.wrapKey.findFirst()
        if (!wrapKey) {
            /**
             * AESの鍵を生成
             * 単純なrandomBytesではなく、createSecretKeyを使って適切なKeyObjectを生成
             */
            const aesKeyBytes = crypto.randomBytes(32); // 256ビット
            const aesKey = crypto.createSecretKey(aesKeyBytes);
            /**
             * KeyObjectからバイナリ形式でエクスポートし、Base64にエンコード
             */
            const exportedKey = aesKey.export();
            const encodeWrapKey = exportedKey.toString('base64');
            await this.prisma.wrapKey.create({
                data: {
                    key: encodeWrapKey
                }
            })
            return encodeWrapKey;
        }
        return wrapKey.key
    }
    async getPrivateKeys(wrapKey: string, kid?: string[]) {
        const privateKeys = await this.prisma.privateKey.findMany({ where: { kid: { in: kid } } });
        const publicKeys = await this.prisma.publicKey.findMany({ where: { kid: { in: kid } } });
        /**
         * 公開鍵と秘密鍵のペアで存在するものだけを取得
         */
        const keys = privateKeys.filter((privateKey) => {
            return publicKeys.some((publicKey) => {
                return publicKey.kid === privateKey.kid
            })
        });
        const exec = keys.map(async (key) => {
            /**
             * 秘密鍵を復号するための情報を取得
             */
            const [ivBase64, encryptedDataBase64] = key.privateKey.split(':');
            const iv = Buffer.from(ivBase64, 'base64');
            const encryptedData = Buffer.from(encryptedDataBase64, 'base64');
            /**
             * AESキーをKeyObjectとして適切にインポート
             */
            const keyBuffer = Buffer.from(wrapKey, 'base64');
            const aesKey = crypto.createSecretKey(keyBuffer);
            /**
             * 暗号化されたデータから認証タグを取り出す（最後の16バイト）
             */
            const authTag = encryptedData.slice(encryptedData.length - 16);
            /**
             * 認証タグを除いた暗号化されたデータを取り出す
             */
            const encryptedPrivateKey = encryptedData.slice(0, encryptedData.length - 16);
            /** 復号用のAES-GCMの鍵を作成 */
            const decipher = crypto.createDecipheriv('aes-256-gcm', aesKey, iv);
            decipher.setAuthTag(authTag);
            /** 復号 */
            const privateKeyBuffer = Buffer.concat([
                decipher.update(encryptedPrivateKey),
                decipher.final()
            ]);
            /** PEM形式の文字列に変換 */
            const privateKeyPem = privateKeyBuffer.toString('utf-8');
            return {
                kid: key.kid,
                privateKey: privateKeyPem
            }
        });
        return Promise.all(exec)
    }
    /**
     * 公開鍵をJWKSの形式でエクスポートできる形に変換
     */
    async exportPublicKeys() {
        const alreadyExistKeys = await this.prisma.publicKey.findMany();
        const keys = alreadyExistKeys.map((key) => {
            return {
                kid: key.kid,
                ...JSON.parse(key.publicKey)
            }
        });
        return { keys }
    }
    /** NodeのCryptoで使用できる鍵の形に変換 */
    async convertPublicWebCryptoKeyToPemKey(publicKey: crypto.JsonWebKey) {
        return crypto.createPublicKey({
            key: publicKey,
            format: 'jwk',
        })
    }
}
```
### NodeのCryptoとWeb Crypto APIを合わせた実装
鍵を作成や暗号化については、Web Crypto APIをメインで行い、Node のCryptoを使った鍵の変換メソッドを設けています。
```tsx
import { PrismaClient } from '@/__generated__';
import { Injectable, InternalServerErrorException } from '@nestjs/common';
import { v4 as uuidv4 } from 'uuid'
import * as crypto from 'crypto';
import { type JsonWebKey } from 'crypto';
const KEY_BASE_OPTIONS = {
    name: 'ECDSA',
    namedCurve: "P-256",
} as const
@Injectable()
export class SignKeyNodeAndWebCryptoService {
    private prisma: PrismaClient
    constructor() {
        this.prisma = new PrismaClient();
    }
    async generateSignKeys() {
        /**
         * ECDSAの公開鍵・秘密鍵のペアを生成
         */
        const keyPair = await crypto.subtle.generateKey(
            {
                ...KEY_BASE_OPTIONS,
            },
            true,
            /**
             * 用途はJWTの署名・検証のためなので、signとverifyを指定
             * https://developer.mozilla.org/ja/docs/Web/API/SubtleCrypto/generateKey#keyusages
             */
            ['sign', 'verify']
        );
        if (keyPair instanceof CryptoKey) {
            throw new InternalServerErrorException('Failed to generate key pair')
        }
        return {
            kid: uuidv4(),
            ...keyPair,
        }
    }
    async saveKeys(kid: string, keys: { publicKey: CryptoKey, privateKey: CryptoKey }): Promise<void> {
        /**
         * 秘密鍵を暗号化するためのAESキーを取得
         */
        let wrapKey = (await this.prisma.wrapKey.findFirst())?.key
        if (!wrapKey) {
            /**
             * 秘密鍵を暗号化するためのAESキーが存在しない場合は作成
             */
            wrapKey = await this.findOrCreateWrapKey();
        }
        /**
         * 公開鍵はJSON形式で提供するためにJWK形式へ変換
         */
        const publicKey = await crypto.subtle.exportKey('jwk', keys.publicKey);
        /**
         * DBへ文字列保存したAESの鍵をCryptoKeyに変換
         * この鍵を使って秘密鍵を暗号化
         * wrapKeyは文字列変換→Base64エンコードされているので、元のArrayBufferに戻してからCryptoKeyに変換する
         */
        const aesKey = await crypto.subtle.importKey('raw', this.convertBase64StringToArrayBufferView(wrapKey), {
            name: 'AES-GCM',
        }, true, ['wrapKey', 'unwrapKey']);
        /**
         * 初期化ベクトルを生成
         * 初期化ベクトルは使い回してはいけないので、毎回生成する
         */
        const iv = crypto.getRandomValues(new Uint8Array(12)).buffer
        /**
         * 秘密鍵をAES-GCMのアルゴリズムで暗号化した上で、pkcs8形式で出力する
         */
        const wrappedPrivateKey = await crypto.subtle.wrapKey('pkcs8', keys.privateKey, aesKey, {
            name: "AES-GCM",
            iv,
        });
        /**
         * 秘密鍵と公開鍵は一緒に保存する必要があるため、エラーが発生した場合は両者を削除
         * Prismaを使用している場合は、通常$transactionで囲めば良い。
         * ただし、今回はCloudflare D1でも使用できることを想定しているため、トランザクションがサポートされていない前提で考える必要がある
         * https://www.prisma.io/docs/orm/overview/databases/cloudflare-d1#limitations
         * そのため、明示的に削除処理を行っている
         */
        try {
            await this.prisma.privateKey.create({
                data: {
                    kid,
                    /**
                     * 初期化ベクトルは秘匿しておく必要はないので、秘密鍵の先頭に付与させる
                     * また、初期化ベクトル・暗号化された秘密鍵はArrayBufferなので文字列として保存しておくためにBase64エンコードする
                     */
                    privateKey: `${this.encodeBase64(iv)}:${this.encodeBase64(wrappedPrivateKey)}`,
                }
            })
            await this.prisma.publicKey.create({
                data: {
                    kid,
                    publicKey: JSON.stringify(publicKey),
                }
            })
        } catch (e) {
            await this.prisma.privateKey.deleteMany({
                where: {
                    kid
                }
            })
            await this.prisma.publicKey.deleteMany({
                where: {
                    kid
                }
            })
            throw e;
        }
    }
    /** AESの鍵を作成し、DBに保存 */
    async findOrCreateWrapKey() {
        const wrapKey = await this.prisma.wrapKey.findFirst()
        if (!wrapKey) {
            /**
             * AESの鍵を生成
             */
            const aesKey = await crypto.subtle.generateKey(
                {
                    name: 'AES-GCM',
                    length: 256,
                },
                true,
                /**
                 * 用途はECDSAの秘密鍵を暗号・複合するためなので、wrapKeyとunwrapKeyを指定
                 * https://developer.mozilla.org/ja/docs/Web/API/SubtleCrypto/generateKey#keyusages
                 */
                ['wrapKey', 'unwrapKey']
            );
            /**
             * DBに保存するためにRawデータに変換
             */
            const exportWrapKey = await crypto.subtle.exportKey('raw', aesKey);
            /**
             *  RawデータをBase64にエンコードし、DBに保存
             */
            const decodeWrapKey = this.encodeBase64(exportWrapKey);
            await this.prisma.wrapKey.create({
                data: {
                    key: decodeWrapKey
                }
            })
            return decodeWrapKey;
        }
        return wrapKey.key
    }
    async getPrivateKeys(wrapKey: string, kid?: string[]) {
        /**
         * 秘密鍵を複合するためのAESの鍵を複合できる形でインポートする
         * この時、wrapKeyはBase64エンコードされているので、デコードしてArrayBufferに変換する
         */
        const importWrapKey = await crypto.subtle.importKey('raw', this.convertBase64StringToArrayBufferView(wrapKey), {
            name: 'AES-GCM',
            length: 256,
        }, true, ['wrapKey', 'unwrapKey']);
        const privateKeys = await this.prisma.privateKey.findMany({ where: { kid: { in: kid } } });
        const publicKeys = await this.prisma.publicKey.findMany({ where: { kid: { in: kid } } });
        /**
         * 公開鍵と秘密鍵のペアで存在するものだけを取得
         */
        const keys = privateKeys.filter((privateKey) => {
            return publicKeys.some((publicKey) => {
                return publicKey.kid === privateKey.kid
            })
        });
        const exec = keys.map(async (key) => {
            /**
             * 秘密鍵を複合するための初期化ベクトルと暗号化された秘密鍵を取得
             * 初期化ベクトルと暗号化された秘密鍵はBase64エンコードされているので、デコードしてArrayBufferに変換する
             */
            const [iv, wrappedPrivateKey] = key.privateKey.split(':').map((str) => this.convertBase64StringToArrayBufferView(str
            ));
            /**
             * 秘密鍵をAES-GCMのアルゴリズムで複合する
             */
            const privateKey = await crypto.subtle.unwrapKey('pkcs8', wrappedPrivateKey, importWrapKey, {
                name: "AES-GCM",
                iv,
            }, KEY_BASE_OPTIONS, true, ['sign']);
            return {
                kid: key.kid,
                privateKey
            }
        });
        return Promise.all(exec)
    }
    /**
     * 公開鍵をJWKSの形式でエクスポートできる形に変換
     */
    async exportPublicKeys() {
        const alreadyExistKeys = await this.prisma.publicKey.findMany();
        const keys = alreadyExistKeys.map((key) => {
            return {
                kid: key.kid,
                ...JSON.parse(key.publicKey) as JsonWebKey
            }
        });
        return { keys }
    }
    /** Web Crypto APIの秘密鍵をNodeのCryptoで使用できる鍵の形に変換 */
    async convertPrivateWebCryptoKeyToPemKey(privateKey: CryptoKey) {
        const exportedPrivateKey = await crypto.subtle.exportKey('pkcs8', privateKey);
        /**ArrayBufferをNode.jsのBufferに変換 */
        const nodeBuffer = Buffer.from(exportedPrivateKey);
        /**PEM形式に変換 */
        const pemKey = `-----BEGIN PRIVATE KEY-----\n${nodeBuffer.toString('base64').match(/.{1,64}/g)?.join('\n')}\n-----END PRIVATE KEY-----\n`;
        return pemKey;
    }
    /** Web Crypto APIの公開鍵をNodeのCryptoで使用できる鍵の形に変換 */
    async convertPublicWebCryptoKeyToPemKey(publicKey: JsonWebKey) {
        return crypto.createPublicKey({
            key: publicKey,
            format: 'jwk',
        })
    }
    /**
     * ArrayBufferをBase64にエンコード
     */
    private encodeBase64(buffer: ArrayBuffer) {
        return btoa(String.fromCharCode(...new Uint8Array(buffer)))
    }
    /**
     * Base64エンコードされた文字列をArrayBufferに変換
     */
    private convertBase64StringToArrayBufferView(str: string) {
        const decodeString = atob(str);
        const buf = new ArrayBuffer(decodeString.length);
        const bufView = new Uint8Array(buf);
        for (let i = 0, strLen = decodeString.length; i < strLen; i++) {
            bufView[i] = decodeString.charCodeAt(i);
        }
        return buf;
    }
}
```
### Web Crypto APIのみで実装
Web Crypto APIのみの場合、鍵以外に署名・検証部分も自分で実装しておく必要があるので、コードとしては二つになります。
まずは、鍵の作成や保存部分についてです。
```tsx
import { PrismaClient } from '@/__generated__';
import { Injectable } from '@nestjs/common';
import { v4 as uuidv4 } from 'uuid'
const KEY_BASE_OPTIONS = {
    name: 'ECDSA',
    namedCurve: "P-256",
    hash: 'SHA-256'
} as const
@Injectable()
export class SignKeyOnlyWebCryptoApiService {
    private prisma: PrismaClient
    constructor() {
        this.prisma = new PrismaClient();
    }
    async generateSignKeys() {
        /**
         * ECDSAの公開鍵・秘密鍵のペアを生成
         */
        const keyPair = await crypto.subtle.generateKey(
            {
                ...KEY_BASE_OPTIONS,
            },
            true,
            /**
             * 用途はJWTの署名・検証のためなので、signとverifyを指定
             * https://developer.mozilla.org/ja/docs/Web/API/SubtleCrypto/generateKey#keyusages
             */
            ['sign', 'verify']
        );
        return {
            kid: uuidv4(),
            ...keyPair,
        }
    }
    async saveKeys(kid: string, keys: { publicKey: CryptoKey, privateKey: CryptoKey }): Promise<void> {
        /**
         * 秘密鍵を暗号化するためのAESキーを取得
         */
        let wrapKey = (await this.prisma.wrapKey.findFirst())?.key
        if (!wrapKey) {
            /**
             * 秘密鍵を暗号化するためのAESキーが存在しない場合は作成
             */
            wrapKey = await this.findOrCreateWrapKey();
        }
        /**
         * 公開鍵はJSON形式で提供するためにJWK形式へ変換
         */
        const publicKey = await crypto.subtle.exportKey('jwk', keys.publicKey);
        /**
         * DBへ文字列保存したAESの鍵をCryptoKeyに変換
         * この鍵を使って秘密鍵を暗号化
         * wrapKeyは文字列変換→Base64エンコードされているので、元のArrayBufferに戻してからCryptoKeyに変換する
         */
        const aesKey = await crypto.subtle.importKey('raw', this.convertBase64StringToArrayBufferView(wrapKey), {
            name: 'AES-GCM',
        }, true, ['wrapKey', 'unwrapKey']);
        /**
         * 初期化ベクトルを生成
         * 初期化ベクトルは使い回してはいけないので、毎回生成する
         */
        const iv = crypto.getRandomValues(new Uint8Array(12)).buffer
        /**
         * 秘密鍵をAES-GCMのアルゴリズムで暗号化した上で、pkcs8形式で出力する
         */
        const wrappedPrivateKey = await crypto.subtle.wrapKey('pkcs8', keys.privateKey, aesKey, {
            name: "AES-GCM",
            iv,
        });
        /**
         * 秘密鍵と公開鍵は一緒に保存する必要があるため、エラーが発生した場合は両者を削除
         * Prismaを使用している場合は、通常$transactionで囲めば良い。
         * ただし、今回はCloudflare D1の使用も想定しているため、トランザクションがサポートされていない前提で考える必要がある
         * https://www.prisma.io/docs/orm/overview/databases/cloudflare-d1#limitations
         * そのため、明示的に削除処理を行っている
         */
        try {
            await this.prisma.privateKey.create({
                data: {
                    kid,
                    /**
                     * 初期化ベクトルは秘匿しておく必要はないので、秘密鍵の先頭に付与させる
                     * また、初期化ベクトル・暗号化された秘密鍵はArrayBufferなので文字列として保存しておくためにBase64エンコードする
                     */
                    privateKey: `${this.encodeBase64(iv)}:${this.encodeBase64(wrappedPrivateKey)}`,
                }
            })
            await this.prisma.publicKey.create({
                data: {
                    kid,
                    publicKey: JSON.stringify(publicKey),
                }
            })
        } catch (e) {
            await this.prisma.privateKey.deleteMany({
                where: {
                    kid
                }
            })
            await this.prisma.publicKey.deleteMany({
                where: {
                    kid
                }
            })
            throw e;
        }
    }
    /** AESの鍵を作成し、DBに保存 */
    async findOrCreateWrapKey() {
        const wrapKey = await this.prisma.wrapKey.findFirst()
        if (!wrapKey) {
            /**
             * AESの鍵を生成
             */
            const aesKey = await crypto.subtle.generateKey(
                {
                    name: 'AES-GCM',
                    length: 256,
                },
                true,
                /**
                 * 用途はECDSAの秘密鍵を暗号・複合するためなので、wrapKeyとunwrapKeyを指定
                 * https://developer.mozilla.org/ja/docs/Web/API/SubtleCrypto/generateKey#keyusages
                 */
                ['wrapKey', 'unwrapKey']
            );
            /**
             * DBに保存するためにRawデータに変換
             */
            const exportWrapKey = await crypto.subtle.exportKey('raw', aesKey);
            /**
             *  RawデータをBase64にエンコードし、DBに保存
             */
            const decodeWrapKey = this.encodeBase64(exportWrapKey);
            await this.prisma.wrapKey.create({
                data: {
                    key: decodeWrapKey
                }
            })
            return decodeWrapKey;
        }
        return wrapKey.key
    }
    async getPrivateKeys(wrapKey: string, kid?: string[]) {
        /**
         * 秘密鍵を複合するためのAESの鍵を複合できる形でインポートする
         * この時、wrapKeyはBase64エンコードされているので、デコードしてArrayBufferに変換する
         */
        const importWrapKey = await crypto.subtle.importKey('raw', this.convertBase64StringToArrayBufferView(wrapKey), {
            name: 'AES-GCM',
            length: 256,
        }, true, ['wrapKey', 'unwrapKey']);
        const privateKeys = await this.prisma.privateKey.findMany({ where: { kid: { in: kid } } });
        const publicKeys = await this.prisma.publicKey.findMany({ where: { kid: { in: kid } } });
        /**
         * 公開鍵と秘密鍵のペアで存在するものだけを取得
         */
        const keys = privateKeys.filter((privateKey) => {
            return publicKeys.some((publicKey) => {
                return publicKey.kid === privateKey.kid
            })
        });
        const exec = keys.map(async (key) => {
            /**
             * 秘密鍵を複合するための初期化ベクトルと暗号化された秘密鍵を取得
             * 初期化ベクトルと暗号化された秘密鍵はBase64エンコードされているので、デコードしてArrayBufferに変換する
             */
            const [iv, wrappedPrivateKey] = key.privateKey.split(':').map((str) => this.convertBase64StringToArrayBufferView(str
            ));
            /**
             * 秘密鍵をAES-GCMのアルゴリズムで複合する
             */
            const privateKey = await crypto.subtle.unwrapKey('pkcs8', wrappedPrivateKey, importWrapKey, {
                name: "AES-GCM",
                iv,
            }, KEY_BASE_OPTIONS, true, ['sign']);
            return {
                kid: key.kid,
                privateKey
            }
        });
        return Promise.all(exec)
    }
    /**
     * 公開鍵をJWKSの形式でエクスポートできる形に変換
     */
    async exportPublicKeys() {
        const alreadyExistKeys = await this.prisma.publicKey.findMany();
        const keys = alreadyExistKeys.map((key) => {
            return {
                kid: key.kid,
                ...JSON.parse(key.publicKey) as JsonWebKey
            }
        });
        return { keys }
    }
    /**
     * JWK形式の公開鍵をCryptoKeyに変換
     */
    async convertPublicWebCryptoKey(publicKey: JsonWebKey) {
        return await crypto.subtle.importKey('jwk', publicKey, KEY_BASE_OPTIONS, true, ['verify']);
    }
    /**
     * ArrayBufferをBase64にエンコード
     */
    private encodeBase64(buffer: ArrayBuffer) {
        return btoa(String.fromCharCode(...new Uint8Array(buffer)))
    }
    /**
     * Base64エンコードされた文字列をArrayBufferに変換
     */
    private convertBase64StringToArrayBufferView(str: string) {
        const decodeString = atob(str);
        const buf = new ArrayBuffer(decodeString.length);
        const bufView = new Uint8Array(buf);
        for (let i = 0, strLen = decodeString.length; i < strLen; i++) {
            bufView[i] = decodeString.charCodeAt(i);
        }
        return buf;
    }
}
```
次に署名・検証部分の処理です。
```tsx
import { Injectable } from "@nestjs/common";
const KEY_BASE_OPTIONS = {
    name: 'ECDSA',
    namedCurve: "P-256",
    hash: 'SHA-256'
} as const
type TokenHeader = {
    alg: 'ES256',
    typ: 'JWT',
    kid: string
}
@Injectable()
export class SignForOnlyWebCryptoApiService {
    async sign(payload: Record<string, unknown>, header: TokenHeader, secretKey: CryptoKey) {
        const encoder = new TextEncoder()
        /**
         * ヘッダーとペイロードをそれぞれURLで使用可能な形式にエンコード
         */
        const payloadEncoded = this.encodeBase64Url(encoder.encode(JSON.stringify(payload)).buffer)
        const headerEncoded = this.encodeBase64Url(encoder.encode(JSON.stringify(header)).buffer)
        /**
         * 署名対象の部分を作成
         */
        const partialToken = `${headerEncoded}.${payloadEncoded}`
        /**
         * 署名を作成
         */
        const signPart = await crypto.subtle.sign(KEY_BASE_OPTIONS, secretKey, encoder.encode(partialToken))
        /**
         * 署名をURLのクエリで使用可能な形式にエンコード
         */
        const signEncoded = this.encodeBase64Url(signPart)
        /**
         * Header.Payload.Signatureの形式でJWSを作成
         */
        return `${partialToken}.${signEncoded}`;
    }
    async verify(token: string, publicKey: CryptoKey) {
        /** JWSの形式を前提に各ブロックを分割する */
        const [header, payload, signature] = token.split('.')
        const encoder = new TextEncoder()
        /**
         * 署名を検証
         * 署名はURLのクエリで使用可能な形式にエンコードされているので元のバイナリに戻す
         * 署名に使用したheaderとpayloadを結合したものははURLのパスで使用可能な形式にエンコード後、
         * TextEncoderのencodeでArrayBufferViewに変換されたものとなっている。
         * なので、検証に使うときは単純にTextEncoderのencodeでArrayBufferViewに変換したものを使う
         */
        const verify = await crypto.subtle.verify(KEY_BASE_OPTIONS, publicKey, this.stringToArrayBuffer(this.base64UrlDecode(signature)), encoder.encode(`${header}.${payload}`))
        return verify;
    }
    private stringToArrayBuffer(str: string) {
        const buffer = new ArrayBuffer(str.length);
        const bufferView = new Uint8Array(buffer);
        for (let i = 0; i < str.length; i++) {
            bufferView[i] = str.charCodeAt(i);
        }
        return buffer;
    }
    private base64UrlDecode(str: string) {
        /**  URLで使用可能な形式にエンコードされた文字列を元のBase64で使用される文字列に置換 */
        const replaceStr = str.replace(/-/g, '+').replace(/_/g, '/');
        /**
         * Base64の文字数が4の倍数になるようにパディング文字列を追加
         * パディング文字列は'='である
         * パディングについての記事
         * https://qiita.com/yagaodekawasu/items/bd8a1db4529cfc921bba
         */
        const padding = Array(replaceStr.length * 8 % 6).map(() => '=').join('');
        return atob(`${replaceStr}${padding}`);
    }
    /**
     * ArrayBufferをURLのパスやクエリでも使用できるURLライクなBase64にエンコード
     */
    private encodeBase64Url(buf: ArrayBufferLike): string {
        return this.encodeBase64(buf).replace(/\/|\+/g, (m) => ({ '/': '_', '+': '-' }[m] ?? m)).replace(/=/g, '')
    }
    /**
     * ArrayBufferをBase64にエンコード
     */
    private encodeBase64(buf: ArrayBufferLike): string {
        let binary = ''
        const bytes = new Uint8Array(buf)
        for (let i = 0, len = bytes.length; i < len; i++) {
            binary += String.fromCharCode(bytes[i])
        }
        return btoa(binary)
    }
}
```
これらを使用した動作確認コードは以下の通りです。
```tsx
import { Controller, Get } from '@nestjs/common';
import { SignKeyNodeAndWebCryptoService } from './sign-key/sign-key-node-and-web-crypto.service';
import * as jsonwebtoken from 'jsonwebtoken';
import { JwtService } from '@nestjs/jwt';
import { SignKeyOnlyNodeCryptoService } from './sign-key/sign-key-only-node-crypto.service';
import { SignKeyOnlyWebCryptoApiService } from './sign-key/sign-key-only-web-crypto-api.service';
import { SignForOnlyWebCryptoApiService } from './sign-key/sign-for-only-web-crypto-api.service';
@Controller()
export class AppController {
  constructor(
    private readonly signKeyNodeAndWebCryptoService: SignKeyNodeAndWebCryptoService,
    private readonly signKeyOnlyNodeCryptoService: SignKeyOnlyNodeCryptoService,
    private readonly signKeyOnlyWebCryptoApiService: SignKeyOnlyWebCryptoApiService,
    private readonly signForOnlyWebCryptoApiService: SignForOnlyWebCryptoApiService,
    private readonly jwtService: JwtService
  ) { }
	/** NodeのCryptoとWeb Crypto APIを使用したサービスの動作確認用 */
  @Get()
  async getHello() {
    const wrapKey = await this.signKeyNodeAndWebCryptoService.findOrCreateWrapKey();
    const alreadyExistKeys = await this.signKeyNodeAndWebCryptoService.getPrivateKeys(wrapKey)
    const keys = [...alreadyExistKeys]
    if (keys.length < 3) {
      const { kid, privateKey, publicKey } = await this.signKeyNodeAndWebCryptoService.generateSignKeys();
      await this.signKeyNodeAndWebCryptoService.saveKeys(kid, { privateKey, publicKey });
      keys.push({ kid, privateKey });
    }
    const { kid, privateKey } = keys[0];
    const signKey = await this.signKeyNodeAndWebCryptoService.convertPrivateWebCryptoKeyToPemKey(privateKey);
		/** 鍵がNodeのCryptoのものになるので、node-jsonwebtokenをラップしているJwtServiceで署名する */
    const token = this.jwtService.sign({ hoga: 1, fuga: 2 }, {
      algorithm: 'ES256',
      keyid: kid,
      privateKey: signKey,
    })
    const publicKeys = await this.signKeyNodeAndWebCryptoService.exportPublicKeys();
    const publicKey = publicKeys.keys.find((key) => key.kid === kid);
    const nodePublicKey = await this.signKeyNodeAndWebCryptoService.convertPublicWebCryptoKeyToPemKey(publicKey);
		/** 鍵がNodeのCryptoのものになるので、node-jsonwebtokenをラップしているJwtServiceで検証する */
    const verify = this.jwtService.verify(token, {
      algorithms: ['ES256'],
      publicKey: nodePublicKey.export({ type: 'spki', format: 'pem' }),
    })
    return {
      token,
      verify,
    }
  }
	/** NodeのCryptoのみを使用したサービスの動作確認用 */
  @Get('2')
  async getHello2() {
    const wrapKey = await this.signKeyOnlyNodeCryptoService.findOrCreateWrapKey();
    const alreadyExistKeys = await this.signKeyOnlyNodeCryptoService.getPrivateKeys(wrapKey)
    const keys = [...alreadyExistKeys]
    if (keys.length < 3) {
      const { kid, privateKey, publicKey } = await this.signKeyOnlyNodeCryptoService.generateSignKeys();
      await this.signKeyOnlyNodeCryptoService.saveKeys(kid, { privateKey, publicKey });
      keys.push({ kid, privateKey });
    }
    const { kid, privateKey } = keys[0];
    const signKey = privateKey;
    /** 鍵がNodeのCryptoのものになるので、node-jsonwebtokenをラップしているJwtServiceで署名する */
    const token = this.jwtService.sign({ hoga: 1, fuga: 2 }, {
      algorithm: 'ES256',
      keyid: kid,
      privateKey: signKey,
      secret: signKey
    })
    const publicKeys = await this.signKeyOnlyNodeCryptoService.exportPublicKeys();
    const publicKey = publicKeys.keys.find((key) => key.kid === kid);
    const nodePublicKey = await this.signKeyOnlyNodeCryptoService.convertPublicWebCryptoKeyToPemKey(publicKey);
		/** 鍵がNodeのCryptoのものになるので、node-jsonwebtokenをラップしているJwtServiceで検証する */
    const verify = this.jwtService.verify(token, {
      algorithms: ['ES256'],
      publicKey: nodePublicKey.export({ type: 'spki', format: 'pem' }),
    })
    return {
      token,
      verify,
    }
  }
	/** Web Crypto APIのみを使用したサービスの動作確認用 */
  @Get('3')
  async getHello3() {
    const wrapKey = await this.signKeyOnlyWebCryptoApiService.findOrCreateWrapKey();
    const alreadyExistKeys = await this.signKeyOnlyWebCryptoApiService.getPrivateKeys(wrapKey)
    const keys = [...alreadyExistKeys]
    if (keys.length < 3) {
      const { kid, privateKey, publicKey } = await this.signKeyOnlyWebCryptoApiService.generateSignKeys();
      await this.signKeyOnlyWebCryptoApiService.saveKeys(kid, { privateKey, publicKey });
      keys.push({ kid, privateKey });
    }
    const { kid, privateKey } = keys[0];
		/** 鍵がWeb Crypto APIのものになるので、独自実装したサービスのメソッドで署名する */
    const token = await this.signForOnlyWebCryptoApiService.sign({ hoga: 1, fuga: 2 }, {
      alg: 'ES256',
      typ: 'JWT',
      kid
    }, privateKey);
    const publicKeys = await this.signKeyOnlyWebCryptoApiService.exportPublicKeys();
    const publicKey = publicKeys.keys.find((key) => key.kid === kid);
    const importPublicKey = await this.signKeyOnlyWebCryptoApiService.convertPublicWebCryptoKey(publicKey);
		/** 鍵がWeb Crypto APIのものになるので、独自実装したサービスのメソッドで検証する */
    const verify = await this.signForOnlyWebCryptoApiService.verify(token, importPublicKey);
    return {
      token,
      verify,
    }
  }
}
```
なお、動作確認用のコードからも分かるように、NestJSで実装して動作確認を行っています。
ただ、Web Crypto APIのみの実装部分はHonoでも動くことは確認しています。
以上の前提はご留意いただきつつ、コードの一部をこの後簡単に確認し、何をしているかを見ていきます。
## 実装概要と設計方針
ここでは署名鍵の管理と署名・検証プロセスの実装について解説します。
最初に展開しコードは、以下の3つの異なるアプローチで実装しています：
1. **Web Crypto APIのみ**: ブラウザやCloudflare Workersなどの環境で動作する実装
2. **Node.js Cryptoのみ**: Node.js環境下で動作する実装
3. **ハイブリッド実装**: 鍵生成はWeb Crypto APIで行い、署名・検証はNode.js Cryptoで行うアプローチ
これらの実装は共通して以下の技術を使用しています：
- **ECDSA（P-256曲線）**: 非対称鍵暗号方式で署名と検証に使用
- **AES-GCM（256ビット）**: 対称鍵暗号方式で秘密鍵の保護に使用
- **JWK（JSON Web Key）**: 鍵の保存形式として使用
- **Prisma ORM**: データベースアクセスに使用

基本的な処理フローは次のとおりです：
1. ECDSA鍵ペア（公開鍵・秘密鍵）を生成
2. 秘密鍵保護用のAES鍵（ラッピングキー）を生成
3. 秘密鍵をAES-GCMで暗号化してデータベースに保存
4. 公開鍵をJWK形式でデータベースに保存
5. 署名時に秘密鍵を復号し、JWTの生成に使用
6. 検証時に公開鍵を取得し、JWTの検証に使用

## 鍵の生成と保存の実装
### Web Crypto APIのみを使用した実装
Web Crypto APIを使用した実装（`SignKeyOnlyWebCryptoApiService`）は、ブラウザやCloudflare Workers環境で動作することを想定しています。
```tsx
async generateSignKeys() {
    const keyPair = await crypto.subtle.generateKey(
        {
            name: 'ECDSA',
            namedCurve: "P-256",
            hash: 'SHA-256'
        },
        true,
        ['sign', 'verify']
    );
    return {
        kid: uuidv4(),
        ...keyPair,
    }
}
```
ここでの特徴は：
- `crypto.subtle.generateKey`を使用して直接CryptoKeyオブジェクトを生成
- 鍵のエクスポート可能性（`extractable: true`）を指定
- 鍵の用途として`sign`と`verify`を明示

この実装では、Web Crypto APIのネイティブな型（CryptoKey）をそのまま活用しています。
### Node.js Cryptoのみを使用した実装
Node.js Cryptoを使用した実装（`SignKeyOnlyNodeCryptoService`）は、NestJSといったNode.jsで動く環境での使用を想定しています。
```tsx
async generateSignKeys() {
    const keyPair = crypto.generateKeyPairSync('ec', {
        namedCurve: "P-256",
        publicKeyEncoding: {
            type: 'spki',
            format: 'pem'
        },
        privateKeyEncoding: {
            type: 'pkcs8',
            format: 'pem'
        }
    });
    const publicKeyObj = crypto.createPublicKey(keyPair.publicKey);
    const publicKeyJwk = publicKeyObj.export({ format: 'jwk' });
    return {
        kid: uuidv4(),
        publicKey: publicKeyJwk,
        privateKey: keyPair.privateKey,
    }
}
```
この実装の特徴は：
- Node.jsのcrypto.generateKeyPairSyncを使用して同期的に鍵ペアを生成
- PEM形式のエンコーディングを指定
- 公開鍵をJWK形式に変換してリターン

ここでは、Node.jsネイティブの鍵形式（PEM）で生成した後、必要に応じてJWK形式に変換するアプローチを取っています。
### ハイブリッド実装
ハイブリッド実装（`SignKeyNodeAndWebCryptoService`）は、Web Crypto APIで鍵を生成し、必要に応じてNode.js Cryptoで使用できる形式に変換します。
```tsx
async convertPrivateWebCryptoKeyToPemKey(privateKey: CryptoKey) {
    const exportedPrivateKey = await crypto.subtle.exportKey('pkcs8', privateKey);
    const nodeBuffer = Buffer.from(exportedPrivateKey);
    const pemKey = `-----BEGIN PRIVATE KEY-----\n${nodeBuffer.toString('base64').match(/.{1,64}/g)?.join('\n')}\n-----END PRIVATE KEY-----\n`;
    return pemKey;
}
async convertPublicWebCryptoKeyToPemKey(publicKey: JsonWebKey) {
    return crypto.createPublicKey({
        key: publicKey,
        format: 'jwk',
    })
}
```
この実装の主な特徴：
- Web Crypto APIで鍵を生成・管理
- 署名や検証のタイミングでNode.js Cryptoで利用可能な形式に変換
- PEM形式とJWK形式の相互変換機能を提供

### 鍵識別子（KID）の管理と利用
すべての実装で共通して、UUIDを使用した鍵識別子（KID）を生成しています。
```tsx
return {
    kid: uuidv4(),
    ...keyPair,
}
```
KIDの主な用途：
1. データベース内の公開鍵と秘密鍵の対応関係を管理
2. JWT署名時に使用した鍵を識別するためのヘッダー情報として使用
3. 複数の鍵ペアをローテーションするための識別子

## 3. 秘密鍵の保護の実装
### ラッピングキー（Wrap Key）の管理手法
すべての実装では、秘密鍵を直接保存せず、AES-GCMで暗号化してから保存しています。
この暗号化に使用するキー（ラッピングキー）の管理も重要な部分だと考えていますので、触れていきます。
```tsx
async findOrCreateWrapKey() {
    const wrapKey = await this.prisma.wrapKey.findFirst()
    if (!wrapKey) {
        /** Web Crypto API版 */
        const aesKey = await crypto.subtle.generateKey(
            {
                name: 'AES-GCM',
                length: 256,
            },
            true,
            ['wrapKey', 'unwrapKey']
        );
        const exportWrapKey = await crypto.subtle.exportKey('raw', aesKey);
        const decodeWrapKey = this.encodeBase64(exportWrapKey);
        await this.prisma.wrapKey.create({
            data: {
                key: decodeWrapKey
            }
        })
        return decodeWrapKey;
    }
    return wrapKey.key
}
```
各実装でのアプローチ：
- **Web Crypto API**: `crypto.subtle.generateKey`でAES-GCM鍵を生成し、raw形式でエクスポート
- **Node.js Crypto**: `crypto.randomBytes`でAES鍵材料を生成し、`createSecretKey`でKeyObjectに変換
- **共通点**: 生成したラッピングキーはBase64エンコードしてデータベースに保存

ラッピングキーは一度だけ生成され、すべての秘密鍵の暗号化に使用されます。
これにより、秘密鍵自体ではなく、ラッピングキーの保護に集中できます。
### 初期化ベクトル（IV）の生成と秘密鍵暗号化フロー
AES-GCMでの暗号化には初期化ベクトル（IV）が必要で、これが毎回ユニークであることが重要です。
Web Crypto API実装での暗号化処理：
```tsx
const iv = crypto.getRandomValues(new Uint8Array(12)).buffer
const wrappedPrivateKey = await crypto.subtle.wrapKey('pkcs8', keys.privateKey, aesKey, {
    name: "AES-GCM",
    iv,
});
```
Node.js Crypto実装での暗号化処理：
```tsx
const iv = crypto.randomBytes(12);
const cipher = crypto.createCipheriv('aes-256-gcm', aesKey, iv);
const encryptedPrivateKey = Buffer.concat([
    cipher.update(Buffer.from(privateKeyPem, 'utf-8')),
    cipher.final()
]);
const authTag = cipher.getAuthTag();
const wrappedPrivateKey = Buffer.concat([encryptedPrivateKey, authTag]);
```
秘密鍵とIVはデータベースに以下の形式で保存：
```tsx
privateKey: `${this.encodeBase64(iv)}:${this.encodeBase64(wrappedPrivateKey)}`,
```
この形式は「IV:暗号化秘密鍵」となっており、復号時にこれらを分離して使用します。
### フォーマット変換処理（JWKとPEM）の実装意図
異なる環境や用途に応じてフォーマット変換が必要になります：
1. **JWK形式**: JSONベースで、特にWeb環境やJWT関連の処理で使用
2. **PEM形式**: テキストベースで、伝統的なSSL/TLS関連や多くのライブラリが対応

ハイブリッド実装では、これらの変換を以下のように行っています：
```tsx
/** CryptoKey → PEM */
async convertPrivateWebCryptoKeyToPemKey(privateKey: CryptoKey) {
    const exportedPrivateKey = await crypto.subtle.exportKey('pkcs8', privateKey);
    const nodeBuffer = Buffer.from(exportedPrivateKey);
    const pemKey = `-----BEGIN PRIVATE KEY-----\n${nodeBuffer.toString('base64').match(/.{1,64}/g)?.join('\n')}\n-----END PRIVATE KEY-----\n`;
    return pemKey;
}
/** JWK → Node.js KeyObject */
async convertPublicWebCryptoKeyToPemKey(publicKey: JsonWebKey) {
    return crypto.createPublicKey({
        key: publicKey,
        format: 'jwk',
    })
}
```
これらの変換により、Web Crypto APIとNode.js Cryptoの両方の利点を活用できます。
## JWT署名プロセスの実装
### Web Crypto APIのみによる署名実装
Web Crypto APIを使用した署名は`SignForOnlyWebCryptoApiService`で実装されています。
```tsx
async sign(payload: Record<string, unknown>, header: TokenHeader, secretKey: CryptoKey) {
    const encoder = new TextEncoder()
    const payloadEncoded = this.encodeBase64Url(encoder.encode(JSON.stringify(payload)).buffer)
    const headerEncoded = this.encodeBase64Url(encoder.encode(JSON.stringify(header)).buffer)
    const partialToken = `${headerEncoded}.${payloadEncoded}`
    const signPart = await crypto.subtle.sign(
        {
            name: 'ECDSA',
            hash: {name: 'SHA-256'},
        },
        secretKey,
        encoder.encode(partialToken)
    )
    const signEncoded = this.encodeBase64Url(signPart)
    return `${partialToken}.${signEncoded}`;
}
```
この実装の特徴：
- `TextEncoder`を使用してJSONを`Uint8Array`に変換
- `crypto.subtle.sign`を使用して署名を生成
- 独自にBase64URLエンコーディングを実装

### Node.js cryptoによる署名実装
Node.js実装では、NestJSの`JwtService`を利用しています（AppControllerの実装から）：
```tsx
const token = this.jwtService.sign({ hoga: 1, fuga: 2 }, {
    algorithm: 'ES256',
    keyid: kid,
    privateKey: signKey,
})
```
ここでは既存のライブラリを活用し、必要なオプションを指定するだけで署名プロセスを簡略化しています。
### ハイブリッド実装での署名処理
ハイブリッド実装では、Web Crypto APIで生成した鍵をNode.js形式に変換してから署名に使用しています：
```tsx
const signKey = await this.signKeyNodeAndWebCryptoService.convertPrivateWebCryptoKeyToPemKey(privateKey);
const token = this.jwtService.sign({ hoga: 1, fuga: 2 }, {
    algorithm: 'ES256',
    keyid: kid,
    privateKey: signKey,
})
```
この方法では、鍵管理はWeb Crypto APIの機能を活用しつつ、署名処理はNode.jsの充実したライブラリを使用できます。
### Base64URLエンコーディングの実装詳細
JWTではBase64URLエンコーディングが使用されますが、これはWeb Crypto API実装で独自に実装されています：
```tsx
private encodeBase64Url(buf: ArrayBufferLike): string {
    return this.encodeBase64(buf)
        .replace(/\/|\+/g, (m) => ({ '/': '_', '+': '-' }[m] ?? m))
        .replace(/=/g, '')
}
private encodeBase64(buf: ArrayBufferLike): string {
    let binary = ''
    const bytes = new Uint8Array(buf)
    for (let i = 0, len = bytes.length; i < len; i++) {
        binary += String.fromCharCode(bytes[i])
    }
    return btoa(binary)
}
```
特徴：
- 通常のBase64から`+`をに、`/`を`_`に置換
- パディング文字（`=`）を削除
- URLセーフな文字列に変換

## 署名検証プロセスの実装
### 公開鍵のエクスポートと変換
JWKSエンドポイントを提供するために公開鍵の管理と取得はを行います：
```tsx
async exportPublicKeys() {
    const alreadyExistKeys = await this.prisma.publicKey.findMany();
    const keys = alreadyExistKeys.map((key) => {
        return {
            kid: key.kid,
            ...JSON.parse(key.publicKey) as JsonWebKey
        }
    });
    return { keys }
}
```
この関数は、JWKS（JSON Web Key Set）形式で公開鍵を提供し、クライアントが署名を検証できるようにします。
### 各実装における検証フローの違い
Web Crypto API実装の検証処理：
```tsx
async verify(token: string, publicKey: CryptoKey) {
    const [header, payload, signature] = token.split('.')
    const encoder = new TextEncoder()
    const verify = await crypto.subtle.verify(
        {
            name: 'ECDSA',
            hash: {name: 'SHA-256'},
        },
        publicKey,
        this.stringToArrayBuffer(this.base64UrlDecode(signature)),
        encoder.encode(`${header}.${payload}`)
    )
    return verify;
}
```
Node.js実装の検証処理（AppControllerより）：
```tsx
const verify = this.jwtService.verify(token, {
    algorithms: ['ES256'],
    publicKey: nodePublicKey.export({ type: 'spki', format: 'pem' }),
})
```
両者の主な違い：
- Web Crypto APIでは低レベルの処理を独自に実装
- Node.js実装では既存のライブラリを活用

### 検証プロセスにおける型変換と互換性確保
公開鍵の変換処理も、検証プロセスでは重要な役割を果たします：
```tsx
async convertPublicWebCryptoKey(publicKey: JsonWebKey) {
    return await crypto.subtle.importKey(
        'jwk',
        publicKey,
        {
            name: 'ECDSA',
            namedCurve: "P-256",
            hash: 'SHA-256'
        },
        true,
        ['verify']
    );
}
```
この関数は、JWK形式の公開鍵をWeb Crypto APIのCryptoKeyオブジェクトに変換し、検証に使用できるようにします。
## 実装例と各アプローチの利点・制約
実装例のどのパターンを選択するかの個人的な考えも軽く触れておきます。
### Web Crypto APIのみの実装：利点と制約
**利点**：
- ブラウザやCloudflare Workersなどの環境で動作可能
- セキュアなコンテキストで実行され、鍵材料の漏洩リスクが低い
- モダンなAPIでPromiseベースの非同期処理が自然に記述できる

**制約**：
- ブラウザ互換性に依存
- 一部の高度な暗号操作が制限されている場合がある
- フォーマット変換のための独自実装が必要になることが多い

### Node.js Cryptoのみの実装：利点と制約
**利点**：
- 豊富なAPIとフォーマットがサポートされている
- サーバーサイドでの広範な使用実績
- 多くのライブラリとの互換性

**制約**：
- ブラウザ環境では使用できない
- APIが少し古く、コールバックベースの関数も多い
- メモリ上に鍵材料が露出する時間が長い可能性がある

### ハイブリッド実装：どのような場合に選択すべきか
**適切なケース**：
- サーバーレス環境（Cloudflare Workersなど）からNode.jsで実装された既存システムとの連携が必要
- Web Crypto APIの鍵管理とNode.js Cryptoの多機能性の両方を活用したい
- 将来的な環境移行（例：サーバーレス→サーバー）を考慮している

**実装上の注意点**：
- 変換処理にコストがかかる
- 二つのAPIの特性を理解する必要がある
- 環境依存の問題に注意が必要

## おわりに
今回はNodeのCryptoやWeb Crypto APIを使用して、ECDSAによる署名・検証を実装しました。
今までNode前提で実装していたのですが、今回その前提では上手く動かない場合があるのを知れてとても勉強になりました。
鍵の作成や署名・検証についても一通り実装できたので満足です。
ただ、概念などはまだまだ理解できていない箇所が多々あるので、引き続き学んでいきます。
ここまで読んでいただきありがとうございました。