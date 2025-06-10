# 算号代码获取

```cpp
struct Rand{
    unsigned char a,b,c[256];
    inline Rand(int _a,int _b,unsigned char *_c):a(_a),b(_b){memcpy(c,_c,sizeof(c));}
    inline unsigned char m(){
        b+=c[++a];
        std::swap(c[a],c[b]);
        return c[(c[a]+c[b])&255];
    }
    inline int operator()(int a){
        if(!a)
            return 0;
        int u=m();
        return u=(u<<8|m())%a;
    }
};
struct Name{
    struct Skill{
        int id,freq;
        inline void init(int _id){
            id=_id,freq=0;
            return;
        }
    };
    Skill nameskill[128];
    unsigned char val[256],namebase[128],namebonus[128],team[256],name[256];
    int namelen,teamlen,bonuslen;
    bool load(char *rawnamein){
        for(int i=0;i<256;++i)
            name[i]=team[i]=0;
        char *namein=cvt(rawnamein);
        namelen=1,teamlen=1,bonuslen=0;
        for(int i=0,f=0;namein[i];++i){
            if(namein[i]=='@')
                f=1;
            else if(!f)
                name[namelen++]=namein[i];
            else
                team[teamlen++]=namein[i];
        }
        if(namelen==1||teamlen==1)
            return false;
        unsigned char s;
        for(int i=0;i<256;++i)
            val[i]=i;
        for(int i=s=0;i<256;++i){
            s+=team[i%teamlen]+val[i];
            std::swap(val[i],val[s]);
        }
        for(int i=0;i<2;++i){
            for(int j=s=0;j<256;++j){
                s+=name[j%namelen]+val[j];
                std::swap(val[j],val[s]);
            }
        }
        for(int i=0;i<256;++i){
            unsigned char m=val[i]*181+160;
            if(m>=89&&m<217)
                namebase[bonuslen++]=m&63;
        }
        memcpy(namebonus,namebase,sizeof(namebase));
        return true;
    }
    void calcprops(int *propbonus){
        int propcnt=0;
        unsigned char r[32];
        memcpy(r,namebonus,sizeof(r));
        for(int i=10;i<31;i+=3){
            std::sort(r+i,r+i+3);
            propbonus[propcnt++]=r[i+1];
        }
        std::sort(r,r+10);
        propbonus[propcnt++]=154;
        for(int i=3;i<7;++i)
            propbonus[propcnt-1]+=r[i];
        return;
    }
    void calcskill(){
        unsigned char *a=namebonus+64,*b=namebase+64;
        Rand rand(0,0,val);
        for(int i=0;i<40;++i)
            nameskill[i].init(i);
        for(int s=0,i=0;i<2;++i){
            for(int j=0;j<40;++j){
                s=(s+rand(40)+nameskill[j].id)%40;
                swap(nameskill[j].id,nameskill[s].id);
            }
        }
        int last=-1;
        for(int i=0,j=0;i<64;i+=4,++j){
            unsigned char p=min({a[i],a[i+1],a[i+2],a[i+3]}),q=min({b[i],b[i+1],b[i+2],b[i+3]});
            if(p>10){
                if(nameskill[j].id<35)
                    nameskill[j].freq=p-10;
                if(q<=10)
                    nameskill[j].e=true;
                else if(nameskill[j].id<25)
                    last=j;
            }
        }
        if(last!=-1){
            nameskill[last].e=true;
            nameskill[last].freq*=2;
        }
        unsigned char u;
        u=nameskill[14].freq;
        if(u>0&&(!nameskill[14].e)){
            nameskill[14].freq+=min({namebonus[60],namebonus[61],u});
            nameskill[14].e=true;
        }
        u=nameskill[15].freq;
        if(u>0&&(!nameskill[15].e)){
            nameskill[15].freq+=min({namebonus[62],namebonus[63],u});
            nameskill[15].e=true;
        }
        return;
    }
};
```
