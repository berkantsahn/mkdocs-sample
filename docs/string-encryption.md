# String Encryption

İlk olarak Volo.Abp.Security paketinde bulunan StringEncryption modülü hatalı olduğu için bu modülün kodlarını ABP GitHup repository’ sinden kodlarını alıp kendimiz oluşturuyoruz.

Bunun için Domain katmanında **AbpStringEncryptionOptions, IStringEncryptionService** ve **StringEncryptionService** adında üç tane sınıf oluşturuyoruz.

## AbpStringEncryptionOptions

* * *

```
using System;
using System.Collections.Generic;
using System.Text;

namespace Serender.Definitions.CustomerMasters
{
    public class AbpStringEncryptionOptions
    {
        public int Keysize { get; set; }
        public string DefaultPassPhrase { get; set; }
        public byte[] InitVectorBytes { get; set; }
        public byte[] DefaultSalt { get; set; }

        public AbpStringEncryptionOptions()
        {
            Keysize = 256;
            DefaultPassPhrase = "gsKnGZ041HLL4IM8";
            InitVectorBytes = Encoding.ASCII.GetBytes("jkE49230Tf093b42");
            DefaultSalt = Encoding.ASCII.GetBytes("hgt!16kl");
        }
    }
}
```

* * *

## IStringEncryptionService

* * *

```
using JetBrains.Annotations;
using System;
using System.Collections.Generic;
using System.Text;

namespace Serender.Definitions.CustomerMasters
{
    public interface IStringEncryptionService
    {
        [CanBeNull]
        string Encrypt([CanBeNull] string plainText, string passPhrase = null, byte[] salt = null);

        [CanBeNull]
        string Decrypt([CanBeNull] string cipherText, string passPhrase = null, byte[] salt = null);
    }
}
```

* * *

## StringEncryptionService

```
using Microsoft.Extensions.Options;
using System;
using System.Collections.Generic;
using System.IO;
using System.Security.Cryptography;
using System.Text;
using Volo.Abp.DependencyInjection;

namespace Serender.Definitions.CustomerMasters
{
    public class StringEncryptionService : IStringEncryptionService, ITransientDependency
    {
        protected AbpStringEncryptionOptions Options { get; }

        public StringEncryptionService(IOptions<AbpStringEncryptionOptions> options)
        {
            Options = options.Value;
        }

        public virtual string Encrypt(string plainText, string passPhrase = null, byte[] salt = null)
        {
            if (plainText == null)
            {
                return null;
            }

            if (passPhrase == null)
            {
                passPhrase = Options.DefaultPassPhrase;
            }

            if (salt == null)
            {
                salt = Options.DefaultSalt;
            }

            var plainTextBytes = Encoding.UTF8.GetBytes(plainText);
            using (var password = new Rfc2898DeriveBytes(passPhrase, salt))
            {
                var keyBytes = password.GetBytes(Options.Keysize / 8);
                using (var symmetricKey = Aes.Create())
                {
                    symmetricKey.Mode = CipherMode.CBC;
               using (var encryptor = symmetricKey.CreateEncryptor(keyBytes, Options.InitVectorBytes))
                    {
                        using (var memoryStream = new MemoryStream())
                        {
          using (var cryptoStream = new CryptoStream(memoryStream, encryptor, CryptoStreamMode.Write))
                            {
                                cryptoStream.Write(plainTextBytes, 0, plainTextBytes.Length);
                                cryptoStream.FlushFinalBlock();
                                var cipherTextBytes = memoryStream.ToArray();
                                return Convert.ToBase64String(cipherTextBytes);
                            }
                        }
                    }
                }
            }
        }


        public virtual string Decrypt(string cipherText, string passPhrase = null, byte[] salt = null)
        {
            if (string.IsNullOrEmpty(cipherText))
            {
                return null;
            }

            if (passPhrase == null)
            {
                passPhrase = Options.DefaultPassPhrase;
            }

            if (salt == null)
            {
                salt = Options.DefaultSalt;
            }

            var cipherTextBytes = Convert.FromBase64String(cipherText);
            using (var password = new Rfc2898DeriveBytes(passPhrase, salt))
            {
                var keyBytes = password.GetBytes(Options.Keysize / 8);
                using (var symmetricKey = Aes.Create())
                {
                    symmetricKey.Mode = CipherMode.CBC;
               using (var decryptor = symmetricKey.CreateDecryptor(keyBytes, Options.InitVectorBytes))
                    {
                        using (var memoryStream = new MemoryStream(cipherTextBytes))
                        {
           using (var cryptoStream = new CryptoStream(memoryStream, decryptor, CryptoStreamMode.Read))
                            {
                                var plainTextBytes = new byte[cipherTextBytes.Length];
                                var totalReadCount = 0;
                                while (totalReadCount < cipherTextBytes.Length)
                                {
                                    var buffer = new byte[cipherTextBytes.Length];
                                    var readCount = cryptoStream.Read(buffer, 0, buffer.Length);
                                    if (readCount == 0)
                                    {
                                        break;
                                    }

                                    for (var i = 0; i < readCount; i++)
                                    {
                                        plainTextBytes[i + totalReadCount] = buffer[i];
                                    }

                                    totalReadCount += readCount;
                                }

                                return Encoding.UTF8.GetString(plainTextBytes, 0, totalReadCount);
                            }
                        }
                    }
                }
            }
        }
    }
}
```

- · StringEncryption modülü tamamladıktan sonra Domain.Shared katmanında Attributes isminde bir klasör oluşturuyoruz.
- · Attributes klasörü içinde **EncryptAttribute** isminde bir sınıf oluşturuyoruz.

## EncryptAttribute

```
using System;
using System.Collections.Generic;
using System.Text;

namespace Serender.Definitions.Attributes
{
    public class EncryptAttribute : System.Attribute
    {

        public EncryptAttribute()
        {

        }
    }
}
```

- CustomerMaster sınıfında şifreleme işlemi yapmak istediğimiz field için bu attribute’ u kullanıyoruz.

```
[Encrypt]
public string Adress { get; set; }
```

- · Otomatik şifreleme işlemi için Application.Contracts katmanında CustomerMaster klasörüne **CustomerMasterEncryptDto** isminde yeni bir DTO ekliyoruz.
    
- · Domain katmanındaki CustomerMaster.cs ‘de olduğu gibi şifrelemek istediğimiz field için EncryptAttribute’ u kullanıyoruz.
    

```
using Serender.Definitions.Attributes;
using System;
using System.Collections.Generic;
using System.Text;
using Volo.Abp.Application.Dtos;

namespace Serender.Definitions.CustomerMasters
{
    public class CustomerMasterEncryptDto : EntityDto<Guid>
    {
        public Guid CustomerTypeId { get; set; }
        public string Code { get; set; }
        public string Name { get; set; }
        public Guid CustomerGroupId { get; set; }
        [Encrypt]
        public string Adress { get; set; }
        public string PostCode { get; set; }
        public string Telephone { get; set; }
        public string Mail { get; set; }
        public string CommercialName { get; set; }
        public string TaxOffice { get; set; }
        public string TaxidNumber { get; set; }
        public Guid ChartofAccountsId { get; set; }
    }
}
```

- · Create metodunda otomatik encryption işlemini kullanabilmek için yeni bir **CustomerMasterEnctyptDtoMapper** isminde yeni bir Mapper sınıfı oluşturuyoruz.
- · Get ve GetList metodunda otomatik decryption işlemini kullanabilmek için yeni bir **CustomCustomerMasterMapper** isminde yeni bir Mapper sınıfı oluşturuyoruz.
- · Update metodunda otomatik encryption işlemini kullanabilmek için yeni bir **CustomerMasterUpdateDtoMapper** isminde yeni bir Mapper sınıfı oluşturuyoruz.
- · Bu Mapper sınıflarında System.Reflection kütüphanesini kullanarak propertyleri alıp source’tan destination’a değerleri taşıyoruz.

## CustomerMasterEnctyptDtoMapper

```
using Serender.Definitions.Attributes;
using Serender.Definitions.CustomerMasters;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using Volo.Abp.DependencyInjection;
using Volo.Abp.ObjectMapping;

namespace Serender.Definitions
{
    public class CustomerMasterEnctyptDtoMapper : IObjectMapper<CreateCustomerMasterDto, CustomerMasterEncryptDto>, ITransientDependency
    {
        protected IStringEncryptionService _stringEncryptionService { get; }

        public CustomerMasterEnctyptDtoMapper(IStringEncryptionService stringEncryptionService)
        {
            _stringEncryptionService = stringEncryptionService;
        }
        public CustomerMasterEncryptDto Map(CreateCustomerMasterDto source)
        {
            var deneme = Map(source, new CustomerMasterEncryptDto());
            return deneme;
        }

        public CustomerMasterEncryptDto Map(CreateCustomerMasterDto source, CustomerMasterEncryptDto destination)
        {

            destination.GetType().GetProperties().ToList().ForEach(dp =>
            {
                if (source.GetType().GetProperties().Any(x => x.Name == dp.Name))
                {
                    if (destination.GetType().GetProperty(dp.Name).GetCustomAttributes(true).Where(sp => sp.GetType() == typeof(EncryptAttribute)).Any())
                    {
                        try
                        {
                            dp.SetValue(destination, _stringEncryptionService.Encrypt(source.GetType().GetProperty(dp.Name).GetValue(source).ToString()));
                        }
                        catch
                        {
                            dp.SetValue(destination, source.GetType().GetProperty(dp.Name).GetValue(source));
                        }
                    }
                    else
                    {
                        dp.SetValue(destination, source.GetType().GetProperty(dp.Name).GetValue(source));
                    }
                }
            }
            );
            return destination;
        }
    }
}
```

## CustomCustomerMasterMapper

```
using Serender.Definitions.Attributes;
using Serender.Definitions.CustomerMasters;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;
using System.Threading.Tasks;
using Volo.Abp.DependencyInjection;
using Volo.Abp.ObjectMapping;

namespace Serender.Definitions
{
    public class CustomCustomerMasterMapper : IObjectMapper<CustomerMaster, CustomerMasterDto>, ITransientDependency
    {
        protected IStringEncryptionService _stringEncryptionService { get; }

        public CustomCustomerMasterMapper(IStringEncryptionService stringEncryptionService)
        {

            _stringEncryptionService = stringEncryptionService;
        }


        public CustomerMasterDto Map(CustomerMaster source)
        {
            var deneme = Map(source, new CustomerMasterDto());
            return deneme;
        }

        public CustomerMasterDto Map(CustomerMaster source, CustomerMasterDto destination)
        {
            destination.GetType().GetProperties().ToList().ForEach(dp =>
            {
                if (source.GetType().GetProperty(dp.Name).GetCustomAttributes(true).Where(sp => sp.GetType() == typeof(EncryptAttribute)).Count() > 0)
                {
                    try
                    {
                        dp.SetValue(destination, _stringEncryptionService.Decrypt(source.GetType().GetProperty(dp.Name).GetValue(source).ToString()));
                    }
                    catch
                    {
                        dp.SetValue(destination, source.GetType().GetProperty(dp.Name).GetValue(source));
                    }
                }
                else
                {
                    dp.SetValue(destination, source.GetType().GetProperty(dp.Name).GetValue(source));
                }
            }
            );

            return destination;

        }
    }
}
```

## CustomerMasterUpdateDtoMapper

```
using AutoMapper;
using Serender.Definitions.CustomerMasters;
using Volo.Abp.DependencyInjection;
using Volo.Abp.ObjectMapping;
using System.Linq;
using System.Reflection;
using System;
using System.Collections.Generic;
using Serender.Definitions.Attributes;

namespace Serender.Definitions
{
    public class CustomerMasterUpdateDtoMapper : IObjectMapper<UpdateCustomerMasterDto, CustomerMasterEncryptDto>, ITransientDependency
    {
        protected IStringEncryptionService _stringEncryptionService { get; }

        public CustomerMasterUpdateDtoMapper(IStringEncryptionService stringEncryptionService)
        {
            _stringEncryptionService = stringEncryptionService;
        }
        public CustomerMasterEncryptDto Map(UpdateCustomerMasterDto source)
        {
            var deneme = Map(source, new CustomerMasterEncryptDto());
            return deneme;
        }

        public CustomerMasterEncryptDto Map(UpdateCustomerMasterDto source, CustomerMasterEncryptDto destination)
        {

            destination.GetType().GetProperties().ToList().ForEach(dp =>
            {
                if (source.GetType().GetProperties().Any(x => x.Name == dp.Name))
                {
                    if (destination.GetType().GetProperty(dp.Name).GetCustomAttributes(true).Where(sp => sp.GetType() == typeof(EncryptAttribute)).Any())
                    {
                        try
                        {
                            dp.SetValue(destination, _stringEncryptionService.Encrypt(source.GetType().GetProperty(dp.Name).GetValue(source).ToString()));
                        }
                        catch
                        {
                            dp.SetValue(destination, source.GetType().GetProperty(dp.Name).GetValue(source));
                        }
                    }
                    else
                    {
                        dp.SetValue(destination, source.GetType().GetProperty(dp.Name).GetValue(source));
                    }
                }
            }
            );
            return destination;
        }
    }
}
```

## Yapılması Gereken Değişiklikler

- Application katmanında bulunan CustomerMasterAppService sınıfı içerine ilk olarak StringEncryption ve ObjectMapper interface’ leri ekliyoruz.

```
public class CustomerMasterAppService : DefinitionsAppService, ICustomerMasterAppService
    { 
  //…
  //…
        private readonly IObjectMapper<DefinitionsApplicationModule> _objectMapper;
        protected IStringEncryptionService _stringEncryptionService { get; }
    public CustomerMasterAppService(
            IStringEncryptionService stringEncryptionService,
            IObjectMapper<DefinitionsApplicationModule> objectMapper)
        {
             //…
//…
            _stringEncryptionService = stringEncryptionService;
            _objectMapper = objectMapper;
        }
}
```

## CreateAsync Metodu

· Oluşturmuş olduğumuz Mapper sınıfına kullanıcıdan gelen input değerlerini gönderiyoruz.

· Mapper ‘ dan dönen değerleri EnctyptDto isminde bir değere atıyoruz.

· CreateAsync için gerekli olan input değerlerini EnctyptDto değişkeninden çekiyoruz.

## GetAsync ve GetListAsync metodu

- Bu iki metot da yapmamız gereken değişiklik sadece return işleminde kendi oluşturduğumuz mapper ‘ ı kullanmak.

```
public async Task<CustomerMasterDto> GetAsync(Guid id)
        {
            var queryable = await _customerMasterRepository.GetQueryableAsync();
            var query = from customerMaster in queryable
                        join customerType in await _customerTypeRepository.GetQueryableAsync() on customerMaster.CustomerTypeId equals customerType.Id
                        join customerGroup in await _customerGroupRepository.GetQueryableAsync() on customerMaster.CustomerGroupId equals customerGroup.Id
                        join accountChart in await _accountChartRepository.GetQueryableAsync() on customerMaster.ChartofAccountsId equals accountChart.Id
                        where customerMaster.Id == id
                        select new { customerMaster, customerType, customerGroup, accountChart };
            var queryResult = await AsyncExecuter.FirstOrDefaultAsync(query);
            if (queryResult == null)
            {
                throw new EntityNotFoundException(typeof(CustomerMaster), id);
            }
       // return ObjectMapper.Map<CustomerMaster, CustomerMasterDto>(queryResult.customerMaster);

            return _objectMapper.Map<CustomerMaster, CustomerMasterDto>(queryResult.customerMaster); 
        }
public async Task<PagedResultDto<CustomerMasterDto>> GetListAsync(GetCustomerMasterListDto input)
        {
            var queryable = await _customerMasterRepository.GetQueryableAsync();

            var query = from customerMaster in queryable
                        join customerType in await _customerTypeRepository.GetQueryableAsync() on customerMaster.CustomerTypeId equals customerType.Id
                        join customerGroup in await _customerGroupRepository.GetQueryableAsync() on customerMaster.CustomerGroupId equals customerGroup.Id
                        join accountChart in await _accountChartRepository.GetQueryableAsync() on customerMaster.ChartofAccountsId equals accountChart.Id
                        select new { customerMaster, customerType, customerGroup, accountChart };

            query = query
                .OrderBy(NormalizeSorting(input.Sorting))
                .Skip(input.SkipCount)
                .Take(input.MaxResultCount);

            var queryResult = await AsyncExecuter.ToListAsync(query);

            var customerMasterDtos = queryResult.Select(x =>
            {
        // return ObjectMapper.Map<CustomerMaster, CustomerMasterDto>(x.customerMaster);
             return _objectMapper.Map<CustomerMaster, CustomerMasterDto>(x.customerMaster);
            }).ToList();

            var totalCount = await _customerMasterRepository.GetCountAsync();

            return new PagedResultDto<CustomerMasterDto>(
                totalCount,
                customerMasterDtos
            );
        }
```

## UpdateAsync metodu

· Bir return’ e ihtiyacımız olduğu için **UpdateAsync** metodunu Task<TResult> yapısına dönüştürüyoruz.

· Aynı işlemi Application.Contracts katmanında bulunan ICustomerMasterAppService sınıfında ve HttpApi katmanında bulunan CustomerMasterController sınıfında da gerçekleştiriyoruz.

· CreateAsync metodunda yaptığımız gibi kullanıcıdan gelen verileri oluşturduğumuz Mapper’ a gönderiyoruz ve bir değişkene atıyoruz.

· Update işlemi için gerekli olan verileri kullanıcıdan gelen input yerine mapper işleminden geri dönen değerler ile gerçekleştiriyoruz.

```
public async Task <CustomerMasterDto> UpdateAsync(Guid id, UpdateCustomerMasterDto input)
        {
            var customerMaster = await _customerMasterRepository.GetAsync(id);
            var EnctyptDto = _objectMapper.Map<UpdateCustomerMasterDto, CustomerMasterEncryptDto>(input);
            if (customerMaster.Code != EnctyptDto.Code || customerMaster.Name != EnctyptDto.Name || customerMaster.CustomerTypeId != EnctyptDto.CustomerTypeId
                || customerMaster.CustomerGroupId != EnctyptDto.CustomerGroupId || customerMaster.Adress != EnctyptDto.Adress || customerMaster.PostCode != EnctyptDto.PostCode
                || customerMaster.Telephone != EnctyptDto.Telephone || customerMaster.Mail != EnctyptDto.Mail || customerMaster.CommercialName != EnctyptDto.CommercialName
                || customerMaster.TaxOffice != EnctyptDto.TaxOffice || customerMaster.TaxidNumber != EnctyptDto.TaxidNumber 
                || customerMaster.ChartofAccountsId != EnctyptDto.ChartofAccountsId)
            {
                await _customerMasterManager.ChangeCodeAsync(customerMaster, EnctyptDto.Code, EnctyptDto.Name, EnctyptDto.CustomerTypeId, EnctyptDto.CustomerGroupId,
                    EnctyptDto.Adress, EnctyptDto.PostCode, EnctyptDto.Telephone, EnctyptDto.Mail, EnctyptDto.CommercialName, EnctyptDto.TaxOffice, EnctyptDto.TaxidNumber,
                    EnctyptDto.ChartofAccountsId);
            }

            await _customerMasterRepository.UpdateAsync(customerMaster);
            return _objectMapper.Map<CustomerMaster, CustomerMasterDto>(customerMaster);
        }
```

```csharp
using System;
using System.Collections.Generic;
using System.Text;

namespace Serender.Definitions.CustomerMasters
{
    public class AbpStringEncryptionOptions
    {
        public int Keysize { get; set; }
        public string DefaultPassPhrase { get; set; }
        public byte[] InitVectorBytes { get; set; }
        public byte[] DefaultSalt { get; set; }

        public AbpStringEncryptionOptions()
        {
            Keysize = 256;
            DefaultPassPhrase = "gsKnGZ041HLL4IM8";
            InitVectorBytes = Encoding.ASCII.GetBytes("jkE49230Tf093b42");
            DefaultSalt = Encoding.ASCII.GetBytes("hgt!16kl");
        }
    }
}
```