# 算号代码获取

```cpp
typedef unsigned long long u64_t;
typedef unsigned char u8_t;
const int N=256;
const int skill_cnt=40;
struct Name
{
    u8_t val[N],ual[N],val_base[N],val_base2[N];
    u8_t name_base[M], freq[16], skill[skill_cnt], p, q;
    int prop[8];
    int q_len,V,seed;
    u8_t m()
    {
        q += val[++p];
        swap(val[p], val[q]);
        return val[val[p] + val[q] & 255];
    }
    int gen()
    {
        int u = m();
        return (u << 8 | m()) % skill_cnt;
    }
    void load_team(char *_team)
    {
        int t_len = strlen(_team)+1;
        u8_t s;
        for (int i = 0; i < N; i++) val_base[i] = i;
        for (int i = s = 0; i < N; ++i)
        {
            if (i % t_len)
                s += _team[i % t_len - 1];
            s += val_base[i];
            swap(val_base[i], val_base[s]);
        }
    }
    void load_name(const char *name)
    {
        memcpy(val, val_base, sizeof val);
        q_len = -1;
        u8_t s;
        int t_len=strlen(name)+1;
        for (int _ = 0; _ < 2; _++)
            for (int i = s = 0; i < N; i++)
            {
                if (i % t_len) s += name[i % t_len - 1];
                s += val[i];
                swap(val[i], val[s]);
            }
        for (int i = 0; i < N; i += 8)
            ual[i] = val[i] * 181 + 160;
        for (int i = 0; i < N; i ++)
            if (ual[i] >= 89 && ual[i] < 217)
                name_base[++q_len] = ual[i] & 63;
        prop[1] = median(name_base[10], name_base[11], name_base[12]);
        prop[2] = median(name_base[13], name_base[14], name_base[15]);
        prop[3] = median(name_base[16], name_base[17], name_base[18]);
        prop[4] = median(name_base[19], name_base[20], name_base[21]);
        prop[5] = median(name_base[22], name_base[23], name_base[24]);
        prop[6] = median(name_base[25], name_base[26], name_base[27]);
        prop[7] = median(name_base[28], name_base[29], name_base[30]);
        sort(name_base, name_base + 10);
        prop[0] = (154 + name_base[3] + name_base[4] + name_base[5] + name_base[6])/ 3;
    }
    void calc_skills(const char *name)
    {
        q_len=-1;
        for (int i = 0; i < N; i += 8)
            ual[i] = val[i] * 181 + 160;
        for (int i = 0; i < N; i ++)
            if (ual[i] >= 89 && ual[i] < 217)
                name_base[++q_len] = ual[i] & 63;

        u8_t *a = name_base + K;
        for (int i = 0; i < skill_cnt; i ++) skill[i] = i;
        memset(freq, 0, sizeof freq);
        p = q = 0;
        for (int s = 0, _ = 0; _ < 2; _ ++)
            for (int i = 0; i < skill_cnt; i ++) {
                s = (s + gen() + skill[i]) % skill_cnt;
                swap(skill[i], skill[s]);
            }
        int last = -1;
        for (int i = 0, j = 0; i < K; i += 4, j ++) {
            u8_t p = min({a[i], a[i + 1], a[i + 2], a[i + 3]});
            if (p > 10 && skill[j] < 35) {
                freq[j] = p - 10;
                if (skill[j] < 25) last = j;
            }
            else freq[j] = 0;
        }
        if (last != -1) freq[last] <<= 1;
        if (freq[14] && last != 14)
            freq[14] += min({name_base[60], name_base[61], freq[14]});
        if (freq[15] && last != 15)
            freq[15] += min({name_base[62], name_base[63], freq[15]});
    }
};
```
