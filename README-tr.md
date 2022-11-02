[<img width="200" alt="get in touch with Consensys Diligence" src="https://user-images.githubusercontent.com/2865694/56826101-91dcf380-685b-11e9-937c-af49c2510aa0.png">](https://consensys.net/diligence/)<br/>
<sup>
[[  🌐  ](https://consensys.net/diligence/)  [  📩  ](mailto:diligence@consensys.net)  [  🔥  ](https://consensys.net/diligence/tools/)]
</sup><br/><br/>


# Smart Contract Security Best Practices
**Notice: this translation was generously provided by a contributor. The maintainers are not able to verify the content. Any issues or PRs to help improve it are welcome.**

Dokumantasyon adresi: https://consensys.github.io/smart-contract-best-practices/

Çince dokumantasyon: https://github.com/ConsenSys/smart-contract-best-practices/blob/master/README-zh.md
Vietnamca dokumantasyon: https://github.com/ConsenSys/smart-contract-best-practices/blob/master/README-vi.md

## Destekleriniz bekleniyor!

Ufak bir düzeltme veya bir yenilik dahi söz konusu olsa, lütfen pull request yapmaktan çekinmeyin.
Eğer yeni bir içerik geliştiriyorsanız lütfen [katkı sağlama](./docs/about/index.md) sayfasını stil amaçlı kontrol edin ve referans verin.

İlgilenilmesi veya güncellenmesi gereken başlıklar için lütfen [sorunlar](https://github.com/ConsenSys/smart-contract-best-practices/issues) sayfasını ziyaret ediniz. Eğer tartışmak istediğiniz bir fikriniz varsa bizimle [Gitter](https://gitter.im/ConsenSys/smart-contract-best-practices) adresinden iletişime geçebilirsiniz.

## Dokumantasyon oluşturulması

```
git clone git@github.com:ConsenSys/smart-contract-best-practices.git
cd smart-contract-best-practices
pip install -r requirements.txt
mkdocs build 
```
Server'ı çalıştırmak için (ve fail sırasında yeniden başlatmak için): 

```
until mkdocs serve; do :; done
```

Siteyi localhost üzerinde görebilmek için `mkdocs serve` komutunu kullanabilirsiniz. Ayrıca her kayıt aldığınız zaman reload olması için de bu komutu kullanabilirsiniz.

## Dokumantasyonu tekrardan yayınlama

```
mkdocs gh-deploy
```
