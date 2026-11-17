<script lang="ts">
  import { onMount, onDestroy } from 'svelte';
  import {
    Trophy,
    Gamepad2,
    Shield,
    Plus,
    X,
    AlertCircle,
    Calendar,
    ChevronRight,
    Swords,
    Users,
    Check,
    XCircle,
    Shuffle,
    Crown,
    LogIn,
    LogOut,
    UserCheck,
    MessageSquare,
    Send,
    DoorOpen,
    User,
    Search,
    Bell,
    Sparkles,
    Reply,
    Trash2,
    Edit3,
    Lock,
    Unlock,
    LayoutDashboard,
    Globe,
    Menu,
    CheckCircle2,
    Mail,
    ShieldAlert
  } from 'lucide-svelte';

  import { supabase } from '../lib/supabase';

  type Role = 'member' | 'leader' | 'admin';
  type Tab =
    | 'hub'
    | 'squads'
    | 'mysquad'
    | 'scrims'
    | 'tournaments'
    | 'members'
    | 'messages'
    | 'allchat'
    | 'dashboard';

  // =========================================================
  // EXCLUSIVE MASTER ADMIN LOCK (ONLY YOUR ACCOUNT CAN BE ADMIN)
  // =========================================================
  const MASTER_ADMIN_ID = '6dedc981-fe9b-45d4-83e5-103e6d607420';
  const MASTER_ADMIN_EMAIL = 'kyalsinshoonlei6@gmail.com';

  let currentRole = $state<Role>('member');
  let activeTab = $state<Tab>('hub');
  let mobileMenuOpen = $state(false);

  // Filters & Search States
  let selectedGameFilter = $state('ALL');
  let squadSearchQuery = $state('');
  let showOpenSquadsOnly = $state(false);
  let tourneyStatusFilter = $state<'ALL' | 'open' | 'ongoing' | 'completed'>('ALL');
  let memberSearchQuery = $state('');
  let memberGameFilter = $state('ALL');

  // Core Data States
  let squads: any[] = $state([]);
  let scrims: any[] = $state([]);
  let tournaments: any[] = $state([]);
  let tournamentParticipants: any[] = $state([]);
  let tournamentMatches: any[] = $state([]);
  let allProfiles: any[] = $state([]);

  // Dashboard Specific States
  let mySquads: any[] = $state([]);
  let pendingMembers: any[] = $state([]);
  let myJoinedSquads: any[] = $state([]);

  // "My Squad" Tab Specific States
  let activeMySquadsList: any[] = $state([]);
  let selectedMySquadId = $state('');
  let squadRoster: any[] = $state([]);
  let squadMessages: any[] = $state([]);
  let newChatMessage = $state('');
  let loadingMySquadRoom = $state(false);
  let editingSquadRequirements = $state(false);
  let updatedRequirementsText = $state('');

  // "Direct Messages" (1-on-1 Player Chat) States
  let directMessages: any[] = $state([]);
  let selectedDmPartnerId = $state('');
  let newDmText = $state('');
  let dmSearchQuery = $state('');
  let dmChannel: any = null;

  // "All Chat" States
  let globalMessages: any[] = $state([]);
  let newGlobalMessage = $state('');
  let replyToMessage: any = $state(null);
  let globalChatChannel: any = null;
  let squadChatChannel: any = null;
  let authListener: any = null;

  // Loading States
  let loadingSquads = $state(true);
  let loadingScrims = $state(true);
  let loadingTournaments = $state(true);
  let loadingDashboard = $state(false);
  let loadingMembers = $state(false);
  let currentUser: any = $state(null);

  // Notification Center & Toast System
  let showNotifications = $state(false);
  let toasts: { id: number; text: string; type: 'success' | 'error' | 'info' }[] = $state([]);
  let toastCounter = 0;

  // Custom Confirmation Modal State
  let confirmDialog = $state<{
    open: boolean;
    title: string;
    message: string;
    onConfirm: () => void;
  }>({
    open: false,
    title: '',
    message: '',
    onConfirm: () => {}
  });

  // Player Profile Modal States
  let showProfileModal = $state(false);
  let profileFullName = $state('');
  let profileStudentId = $state('');
  let profileMlbbId = $state('');
  let profilePubgId = $state('');
  let profileFootballPos = $state('');

  // Create Squad Modal State
  let showCreateModal = $state(false);
  let squadName = $state('');
  let gameType = $state('MLBB');
  let requirements = $state('');

  // Scrim Modal State
  let showScrimModal = $state(false);
  let challengerSquadId = $state('');
  let opponentSquadId = $state('');
  let scheduledTime = $state('');

  // Tournament Modal State (Admin)
  let showTourneyModal = $state(false);
  let tourneyTitle = $state('');
  let tourneyGameType = $state('MLBB');
  let tourneyDeadline = $state('');
  let selectedSquadForTourney: Record<string, string> = $state({});

  // Auth Modal State
  let showAuthModal = $state(false);
  let isLoginMode = $state(true);
  let authEmail = $state('');
  let authPassword = $state('');
  let authFullName = $state('');
  let authStudentId = $state('');

  let loadingAction = $state(false);
  let errorMessage = $state('');
  let successMessage = $state('');

  const quickEmojis = ['🔥', '⚔️', '🏆', 'GG', '👍', '🎮'];

  // ==========================================
  // STRICT SINGLE-ADMIN + MULTI-ROLE STATES
  // ==========================================
  let isSystemAdmin = $derived(
    Boolean(
      currentUser &&
        (currentUser.id === MASTER_ADMIN_ID ||
          currentUser.email?.toLowerCase() === MASTER_ADMIN_EMAIL.toLowerCase())
    )
  );
  let isAnySquadLeader = $derived(mySquads.length > 0);
  let isAnySquadMember = $derived(
    myJoinedSquads.some((m) => m.status === 'approved')
  );

  onMount(async () => {
    await checkUserAndRole();

    const { data: listener } = supabase.auth.onAuthStateChange(async (_event, session) => {
      currentUser = session?.user ?? null;
      if (currentUser) {
        await checkUserAndRole();
        await Promise.all([fetchDashboardData(), fetchDirectMessages()]);
      } else {
        currentRole = 'member';
      }
    });
    authListener = listener;

    await Promise.all([
      fetchSquads(),
      fetchScrims(),
      fetchTournaments(),
      fetchDashboardData(),
      fetchAllMembers(),
      fetchGlobalChat(),
      fetchDirectMessages()
    ]);
    setupRealtimeSubscriptions();
  });

  onDestroy(() => {
    if (globalChatChannel) supabase.removeChannel(globalChatChannel);
    if (squadChatChannel) supabase.removeChannel(squadChatChannel);
    if (dmChannel) supabase.removeChannel(dmChannel);
    if (authListener?.subscription) authListener.subscription.unsubscribe();
  });

  // ==========================================
  // 0. TOASTS, CONFIRM DIALOG & REALTIME
  // ==========================================
  function showToast(text: string, type: 'success' | 'error' | 'info' = 'success') {
    const id = ++toastCounter;
    toasts = [...toasts, { id, text, type }];
    setTimeout(() => {
      toasts = toasts.filter((t) => t.id !== id);
    }, 4200);
  }

  function askConfirm(title: string, message: string, onConfirm: () => void) {
    confirmDialog = { open: true, title, message, onConfirm };
  }

  function getUniquePlayerTag(playerObj: any) {
    if (!playerObj) return '';
    if (playerObj.student_id) return playerObj.student_id;
    if (playerObj.id) return `#${playerObj.id.slice(0, 4).toUpperCase()}`;
    return '';
  }

  // Only show ADMIN badge for the 1 true Master Admin ID
  function getPlayerRoleBadges(playerObj: any) {
    const badges: { label: string; classes: string }[] = [];
    if (playerObj.id === MASTER_ADMIN_ID) {
      badges.push({
        label: '👑 ADMIN',
        classes: 'bg-amber-500/20 text-amber-300 border-amber-500/40'
      });
    }
    const leadsSquad = squads.some((s) => s.leader_id === playerObj.id);
    if (leadsSquad || playerObj.role === 'leader') {
      badges.push({
        label: '🛡️ LEADER',
        classes: 'bg-purple-500/20 text-purple-300 border-purple-500/40'
      });
    }
    if (badges.length === 0) {
      badges.push({
        label: '🎮 MEMBER',
        classes: 'bg-emerald-500/15 text-emerald-300 border-emerald-500/30'
      });
    }
    return badges;
  }

  async function isStudentIdTaken(studentIdStr: string, excludeUserId?: string): Promise<boolean> {
    const normalized = studentIdStr.trim().toUpperCase();
    if (!normalized) return false;

    let query = supabase
      .from('profiles')
      .select('id')
      .ilike('student_id', normalized);

    if (excludeUserId) {
      query = query.neq('id', excludeUserId);
    }

    const { data } = await query;
    return (data && data.length > 0) || false;
  }

  function setupRealtimeSubscriptions() {
    squadChatChannel = supabase
      .channel('squad-room-changes')
      .on(
        'postgres_changes',
        { event: 'INSERT', schema: 'public', table: 'squad_messages' },
        (payload) => {
          if (payload.new && payload.new.squad_id === selectedMySquadId) {
            fetchMySquadDetails(selectedMySquadId);
          }
        }
      )
      .subscribe();

    globalChatChannel = supabase
      .channel('global-chat-changes')
      .on(
        'postgres_changes',
        { event: 'INSERT', schema: 'public', table: 'global_messages' },
        () => {
          fetchGlobalChat();
        }
      )
      .subscribe();

    dmChannel = supabase
      .channel('direct-messages-changes')
      .on(
        'postgres_changes',
        { event: 'INSERT', schema: 'public', table: 'direct_messages' },
        (payload) => {
          if (
            currentUser &&
            payload.new &&
            (payload.new.sender_id === currentUser.id ||
              payload.new.receiver_id === currentUser.id)
          ) {
            fetchDirectMessages();
          }
        }
      )
      .subscribe();
  }

  // ==========================================
  // 0.5 ONE SQUAD PER GAME VALIDATION HELPERS
  // ==========================================
  function getActiveSquadForGame(gType: string) {
    const led = mySquads.find((s) => s.game_type === gType);
    if (led) return led;

    const joined = myJoinedSquads.find(
      (m) => m.status === 'approved' && m.squads?.game_type === gType
    );
    if (joined) return joined.squads;

    return null;
  }

  function getPendingRequestForGame(gType: string) {
    return myJoinedSquads.find(
      (m) => m.status === 'pending' && m.squads?.game_type === gType
    );
  }

  // ==========================================
  // 1. AUTHENTICATION, PROFILE & STRICT ADMIN LOCK
  // ==========================================
  async function checkUserAndRole() {
    try {
      const { data: sessionData } = await supabase.auth.getSession();
      let user = sessionData?.session?.user || null;

      if (!user) {
        const { data: userData } = await supabase.auth.getUser();
        user = userData?.user || null;
      }

      currentUser = user;

      if (!user) {
        currentRole = 'member';
        return;
      }

      const isMaster =
        user.id === MASTER_ADMIN_ID ||
        user.email?.toLowerCase() === MASTER_ADMIN_EMAIL.toLowerCase();

      profileFullName =
        user.user_metadata?.full_name || user.email?.split('@')[0] || 'UIT Player';

      let isAlreadyLeader = false;
      try {
        const { data: ledSquads } = await supabase
          .from('squads')
          .select('id')
          .eq('leader_id', user.id);
        isAlreadyLeader = (ledSquads && ledSquads.length > 0) || false;
      } catch (e) {
        console.warn('Squad check warning:', e);
      }

      try {
        const { data: profile } = await supabase
          .from('profiles')
          .select('*')
          .eq('id', user.id)
          .maybeSingle();

        if (profile) {
          profileFullName =
            profile.full_name || user.user_metadata?.full_name || 'UIT Player';
          profileStudentId = profile.student_id || '';
          profileMlbbId = profile.mlbb_id || '';
          profilePubgId = profile.pubg_id || '';
          profileFootballPos = profile.football_position || '';

          if (isMaster) {
            currentRole = 'admin';
            if (profile.role !== 'admin') {
              await supabase.from('profiles').update({ role: 'admin' }).eq('id', user.id);
            }
          } else {
            // Non-master accounts can NEVER be 'admin'
            const properRole: Role = isAlreadyLeader ? 'leader' : 'member';
            currentRole = properRole;
            if (profile.role !== properRole) {
              await supabase.from('profiles').update({ role: properRole }).eq('id', user.id);
            }
          }
        } else {
          const assignedRole: Role = isMaster
            ? 'admin'
            : isAlreadyLeader
              ? 'leader'
              : 'member';
          currentRole = assignedRole;

          await supabase.from('profiles').upsert({
            id: user.id,
            full_name: profileFullName,
            role: assignedRole
          });
        }
      } catch (e) {
        console.warn('Profile check warning:', e);
        currentRole = isMaster ? 'admin' : isAlreadyLeader ? 'leader' : 'member';
      }
    } catch (err) {
      console.error('Error checking user:', err);
    }
  }

  async function handleSaveProfile(e: SubmitEvent) {
    e.preventDefault();
    if (!currentUser) return;

    loadingAction = true;
    errorMessage = '';
    successMessage = '';

    try {
      const normalizedStudentId = profileStudentId.trim().toUpperCase();

      if (normalizedStudentId) {
        const taken = await isStudentIdTaken(normalizedStudentId, currentUser.id);
        if (taken) {
          errorMessage = `Student ID "${normalizedStudentId}" ကို အခြားကစားသမားတစ်ဦး အသုံးပြုထားပြီးသား ဖြစ်ပါသည်။ တူညီ၍ မရပါ။`;
          loadingAction = false;
          return;
        }
      }

      const updatedRole: Role = isSystemAdmin
        ? 'admin'
        : mySquads.length > 0
          ? 'leader'
          : 'member';

      const { error } = await supabase.from('profiles').upsert({
        id: currentUser.id,
        full_name: profileFullName.trim() || 'UIT Player',
        student_id: normalizedStudentId || null,
        mlbb_id: profileMlbbId.trim() || null,
        pubg_id: profilePubgId.trim() || null,
        football_position: profileFootballPos.trim() || null,
        role: updatedRole
      });

      if (error) {
        if (error.message?.includes('unique') || error.code === '23505') {
          throw new Error(
            `Student ID "${normalizedStudentId}" ကို အခြားကစားသမားတစ်ဦး အသုံးပြုထားပြီးသား ဖြစ်ပါသည်။`
          );
        }
        throw error;
      }

      profileStudentId = normalizedStudentId;
      currentRole = updatedRole;
      successMessage = 'Profile အချက်အလက်များကို သိမ်းဆည်းပြီးပါပြီ!';
      showToast('Player profile updated!', 'success');
      await Promise.all([fetchSquads(), fetchDashboardData(), fetchAllMembers()]);
    } catch (err: any) {
      errorMessage = err.message || 'Profile သိမ်းဆည်းရာတွင် အမှားရှိပါသည်။';
    } finally {
      loadingAction = false;
    }
  }

  async function handleAuth(e: SubmitEvent) {
    e.preventDefault();
    loadingAction = true;
    errorMessage = '';
    successMessage = '';

    try {
      if (isLoginMode) {
        const { data, error } = await supabase.auth.signInWithPassword({
          email: authEmail.trim(),
          password: authPassword
        });
        if (error) throw error;

        currentUser = data.user;
        showAuthModal = false;
        authEmail = '';
        authPassword = '';
        showToast('Signed in successfully!', 'success');
        await checkUserAndRole();
        await Promise.all([fetchDashboardData(), fetchDirectMessages()]);
      } else {
        const normalizedSid = authStudentId.trim().toUpperCase();

        if (normalizedSid) {
          const taken = await isStudentIdTaken(normalizedSid);
          if (taken) {
            errorMessage = `Student ID "${normalizedSid}" ကို အခြားကျောင်းသားတစ်ဦး အသုံးပြုထားပြီးသား ဖြစ်ပါသည်။ မတူညီသော ID ထည့်သွင်းပါ။`;
            loadingAction = false;
            return;
          }
        }

        const { data, error } = await supabase.auth.signUp({
          email: authEmail.trim(),
          password: authPassword,
          options: {
            data: { full_name: authFullName.trim() || 'UIT Player' }
          }
        });
        if (error) throw error;

        if (data.user) {
          const initialRole: Role =
            data.user.id === MASTER_ADMIN_ID ||
            data.user.email?.toLowerCase() === MASTER_ADMIN_EMAIL.toLowerCase()
              ? 'admin'
              : 'member';
          try {
            await supabase.from('profiles').upsert({
              id: data.user.id,
              full_name: authFullName.trim() || 'UIT Player',
              student_id: normalizedSid || null,
              role: initialRole
            });
          } catch (profileErr) {
            console.warn('Initial profile upsert note:', profileErr);
          }
        }

        if (data.user && !data.session) {
          successMessage =
            'အကောင့်ဖွင့်ခြင်း အောင်မြင်ပါသည်! သင့် Email (Inbox သို့မဟုတ် Spam) သို့ ပို့ထားသော Verification Link ကို နှိပ်ပြီးမှ Sign In ပြန်ဝင်ပေးပါ။';
          authPassword = '';
          isLoginMode = true;
        } else {
          currentUser = data.user;
          showAuthModal = false;
          authEmail = '';
          authPassword = '';
          showToast('Student account created!', 'success');
          await checkUserAndRole();
          await Promise.all([fetchDashboardData(), fetchAllMembers(), fetchDirectMessages()]);
        }
      }
    } catch (err: any) {
      if (err.message?.includes('Email not confirmed')) {
        errorMessage =
          'သင့် Email ကို အတည်မပြုရသေးပါ။ Email ထဲသို့ ပို့ထားသော Verification Link ကို အရင်နှိပ်ပေးပါ။';
      } else if (err.message?.includes('Invalid login credentials')) {
        errorMessage = 'Email သို့မဟုတ် Password မှားယွင်းနေပါသည်။ ပြန်လည်စစ်ဆေးပေးပါ။';
      } else {
        errorMessage = err.message || 'Authentication failed.';
      }
    } finally {
      loadingAction = false;
    }
  }

  async function handleSignOut() {
    await supabase.auth.signOut();
    currentUser = null;
    currentRole = 'member';
    mySquads = [];
    pendingMembers = [];
    myJoinedSquads = [];
    activeMySquadsList = [];
    selectedMySquadId = '';
    squadRoster = [];
    squadMessages = [];
    directMessages = [];
    selectedDmPartnerId = '';
    showToast('Signed out successfully.', 'info');
  }

  // ==========================================
  // 2. DATA FETCHING FUNCTIONS
  // ==========================================
  async function fetchSquads() {
    loadingSquads = true;
    try {
      let query = supabase
        .from('squads')
        .select('*, profiles:leader_id(full_name, student_id)')
        .order('created_at', { ascending: false });

      if (selectedGameFilter !== 'ALL') {
        query = query.eq('game_type', selectedGameFilter);
      }

      const { data, error } = await query;
      if (error) throw error;
      squads = data || [];
    } catch (err: any) {
      console.error('Error fetching squads:', err);
    } finally {
      loadingSquads = false;
    }
  }

  async function fetchScrims() {
    loadingScrims = true;
    try {
      const { data, error } = await supabase
        .from('scrim_matches')
        .select(
          '*, challenger:challenger_squad_id(id, name, game_type, leader_id, wins, losses), opponent:opponent_squad_id(id, name, leader_id, wins, losses)'
        )
        .order('created_at', { ascending: false });

      if (error) throw error;
      scrims = data || [];
    } catch (err: any) {
      console.error('Error fetching scrims:', err);
    } finally {
      loadingScrims = false;
    }
  }

  async function fetchTournaments() {
    loadingTournaments = true;
    try {
      const { data: tourneys, error: tErr } = await supabase
        .from('tournaments')
        .select('*, champion:champion_squad_id(name)')
        .order('created_at', { ascending: false });

      if (tErr) throw tErr;
      tournaments = tourneys || [];

      const { data: parts } = await supabase
        .from('tournament_participants')
        .select('*, squad:squad_id(id, name, game_type)');
      tournamentParticipants = parts || [];

      const { data: matches } = await supabase
        .from('tournament_matches')
        .select(
          '*, squad1:squad1_id(id, name), squad2:squad2_id(id, name), winner:winner_id(id, name)'
        )
        .order('round_number', { ascending: true });
      tournamentMatches = matches || [];
    } catch (err: any) {
      console.error('Error fetching tournaments:', err);
    } finally {
      loadingTournaments = false;
    }
  }

  async function fetchAllMembers() {
    loadingMembers = true;
    try {
      const { data, error } = await supabase
        .from('profiles')
        .select('*')
        .order('created_at', { ascending: false });
      if (!error && data) {
        allProfiles = data;
      }
    } catch (err) {
      console.error('Error fetching members:', err);
    } finally {
      loadingMembers = false;
    }
  }

  async function fetchGlobalChat() {
    try {
      const { data, error } = await supabase
        .from('global_messages')
        .select('*, profiles:user_id(full_name, student_id, role)')
        .order('created_at', { ascending: true })
        .limit(100);
      if (!error && data) {
        globalMessages = data;
      }
    } catch {
      // Safe fallback
    }
  }

  // ==========================================
  // 2.2 DIRECT MESSAGES (1-ON-1 PLAYER CHAT)
  // ==========================================
  async function fetchDirectMessages() {
    if (!currentUser) return;
    try {
      const { data, error } = await supabase
        .from('direct_messages')
        .select(
          '*, sender:sender_id(id, full_name, student_id, role), receiver:receiver_id(id, full_name, student_id, role)'
        )
        .or(`sender_id.eq.${currentUser.id},receiver_id.eq.${currentUser.id}`)
        .order('created_at', { ascending: true });

      if (!error && data) {
        directMessages = data;
      }
    } catch {
      // Safe fallback
    }
  }

  function startDirectChatWith(targetPlayerId: string) {
    if (!currentUser) {
      showAuthModal = true;
      return;
    }
    if (targetPlayerId === currentUser.id) {
      showToast('မိမိအကောင့်သို့ ပြန်စာပို့၍ မရပါ။', 'info');
      return;
    }
    selectedDmPartnerId = targetPlayerId;
    activeTab = 'messages';
    fetchDirectMessages();
  }

  async function handleSendDirectMessage(e: SubmitEvent) {
    e.preventDefault();
    if (!currentUser || !selectedDmPartnerId || !newDmText.trim()) return;

    const text = newDmText.trim();
    newDmText = '';

    try {
      const { error } = await supabase.from('direct_messages').insert({
        sender_id: currentUser.id,
        receiver_id: selectedDmPartnerId,
        content: text
      });

      if (error) throw error;
      await fetchDirectMessages();
    } catch {
      const partnerObj = allProfiles.find((p) => p.id === selectedDmPartnerId);
      directMessages = [
        ...directMessages,
        {
          id: crypto.randomUUID(),
          sender_id: currentUser.id,
          receiver_id: selectedDmPartnerId,
          content: text,
          created_at: new Date().toISOString(),
          sender: { id: currentUser.id, full_name: profileFullName },
          receiver: partnerObj
        }
      ];
    }
  }

  async function fetchDashboardData() {
    if (!currentUser) return;
    loadingDashboard = true;

    try {
      const { data: ledSquads } = await supabase
        .from('squads')
        .select('*, profiles:leader_id(full_name, student_id)')
        .eq('leader_id', currentUser.id);

      mySquads = ledSquads || [];

      if (mySquads.length > 0 && !isSystemAdmin) {
        currentRole = 'leader';
      }

      if (mySquads.length > 0) {
        const squadIds = mySquads.map((s) => s.id);
        const { data: members } = await supabase
          .from('squad_members')
          .select(
            '*, profiles:user_id(id, full_name, student_id, mlbb_id, pubg_id, football_position), squads:squad_id(name, game_type)'
          )
          .in('squad_id', squadIds)
          .order('created_at', { ascending: false });

        pendingMembers = members || [];
      } else {
        pendingMembers = [];
      }

      const { data: joined } = await supabase
        .from('squad_members')
        .select('*, squads:squad_id(*, profiles:leader_id(full_name, student_id))')
        .eq('user_id', currentUser.id);

      myJoinedSquads = joined || [];

      const approvedMemberSquads = (joined || [])
        .filter((item) => item.status === 'approved' && item.squads)
        .map((item) => item.squads);

      const combinedMap = new Map();
      for (const s of mySquads) combinedMap.set(s.id, s);
      for (const s of approvedMemberSquads) combinedMap.set(s.id, s);

      activeMySquadsList = Array.from(combinedMap.values());

      if (activeMySquadsList.length > 0) {
        const stillExists = activeMySquadsList.some((s) => s.id === selectedMySquadId);
        if (!selectedMySquadId || !stillExists) {
          selectedMySquadId = activeMySquadsList[0].id;
        }
        await fetchMySquadDetails(selectedMySquadId);
      } else {
        selectedMySquadId = '';
        squadRoster = [];
        squadMessages = [];
      }
    } catch (err) {
      console.error('Error loading dashboard:', err);
    } finally {
      loadingDashboard = false;
    }
  }

  // ==========================================
  // 2.5 "MY SQUAD" ROOM: ROSTER, LEADER CONTROLS & CHAT
  // ==========================================
  async function fetchMySquadDetails(squadId: string) {
    if (!squadId) return;
    loadingMySquadRoom = true;
    editingSquadRequirements = false;

    try {
      const { data: members } = await supabase
        .from('squad_members')
        .select('*, profiles:user_id(id, full_name, student_id, mlbb_id, pubg_id, football_position)')
        .eq('squad_id', squadId)
        .eq('status', 'approved')
        .order('created_at', { ascending: true });

      squadRoster = members || [];

      const { data: msgs } = await supabase
        .from('squad_messages')
        .select('*, profiles:user_id(full_name, student_id)')
        .eq('squad_id', squadId)
        .order('created_at', { ascending: true });

      squadMessages = msgs || [];
    } catch (err) {
      console.error('Error fetching My Squad room details:', err);
    } finally {
      loadingMySquadRoom = false;
    }
  }

  async function handleToggleRecruitment(squadObj: any) {
    try {
      const nextState = !squadObj.is_recruiting;
      const { error } = await supabase
        .from('squads')
        .update({ is_recruiting: nextState })
        .eq('id', squadObj.id);

      if (error) throw error;
      showToast(
        nextState ? 'Squad recruitment opened!' : 'Squad recruitment closed.',
        'info'
      );
      await Promise.all([fetchSquads(), fetchDashboardData()]);
    } catch (err: any) {
      showToast('Error updating recruitment: ' + err.message, 'error');
    }
  }

  async function handleSaveSquadRequirements(squadId: string) {
    if (!updatedRequirementsText.trim()) return;
    try {
      const { error } = await supabase
        .from('squads')
        .update({ requirements: updatedRequirementsText.trim() })
        .eq('id', squadId);

      if (error) throw error;
      editingSquadRequirements = false;
      showToast('Squad requirements updated!', 'success');
      await Promise.all([fetchSquads(), fetchDashboardData()]);
    } catch (err: any) {
      showToast('Error updating requirements: ' + err.message, 'error');
    }
  }

  function handleDisbandSquad(squadId: string, squadNameStr: string) {
    askConfirm(
      'Disband Squad',
      `Are you sure you want to permanently disband "${squadNameStr}"? All roster and match records for this squad will be removed.`,
      async () => {
        try {
          const { error } = await supabase.from('squads').delete().eq('id', squadId);
          if (error) throw error;
          showToast(`Squad "${squadNameStr}" has been disbanded.`, 'info');
          selectedMySquadId = '';
          await Promise.all([fetchSquads(), fetchDashboardData()]);
        } catch (err: any) {
          showToast('Error disbanding squad: ' + err.message, 'error');
        }
      }
    );
  }

  async function handleSendChatMessage(e: SubmitEvent) {
    e.preventDefault();
    if (!currentUser || !selectedMySquadId || !newChatMessage.trim()) return;

    const text = newChatMessage.trim();
    newChatMessage = '';

    try {
      const { error } = await supabase.from('squad_messages').insert({
        squad_id: selectedMySquadId,
        user_id: currentUser.id,
        content: text
      });

      if (error) throw error;
      await fetchMySquadDetails(selectedMySquadId);
    } catch (err: any) {
      showToast('Message ပို့ရာတွင် အမှားရှိပါသည်: ' + err.message, 'error');
    }
  }

  async function handleSendGlobalMessage(e: SubmitEvent) {
    e.preventDefault();
    if (!currentUser) {
      showAuthModal = true;
      return;
    }
    if (!newGlobalMessage.trim()) return;

    const text = replyToMessage
      ? `@${replyToMessage.profiles?.full_name || 'Player'}: ${newGlobalMessage.trim()}`
      : newGlobalMessage.trim();

    newGlobalMessage = '';
    replyToMessage = null;

    try {
      const { error } = await supabase.from('global_messages').insert({
        user_id: currentUser.id,
        content: text
      });

      if (error) throw error;
      await fetchGlobalChat();
    } catch {
      globalMessages = [
        ...globalMessages,
        {
          id: crypto.randomUUID(),
          user_id: currentUser.id,
          content: text,
          created_at: new Date().toISOString(),
          profiles: { full_name: profileFullName || 'UIT Player', role: currentRole }
        }
      ];
    }
  }

  function handleLeaveSquad(squadId: string, squadNameStr: string) {
    if (!currentUser) return;
    askConfirm(
      'Leave Squad',
      `"${squadNameStr}" အသင်းမှ ထွက်ခွာရန် သေချာပါသလား?`,
      async () => {
        try {
          const { error } = await supabase
            .from('squad_members')
            .delete()
            .eq('squad_id', squadId)
            .eq('user_id', currentUser.id);

          if (error) throw error;

          showToast(`Left "${squadNameStr}" successfully.`, 'info');
          await fetchDashboardData();
        } catch (err: any) {
          showToast('အသင်းမှ ထွက်ရာတွင် အမှားရှိပါသည်: ' + err.message, 'error');
        }
      }
    );
  }

  function handleRemoveMember(memberRowId: string, memberName: string) {
    askConfirm(
      'Kick Squad Member',
      `${memberName} ကို အသင်းမှ ထုတ်ပယ်ရန် သေချာပါသလား?`,
      async () => {
        try {
          const { error } = await supabase
            .from('squad_members')
            .delete()
            .eq('id', memberRowId);

          if (error) throw error;
          showToast(`${memberName} removed from squad.`, 'info');
          await fetchMySquadDetails(selectedMySquadId);
          await fetchDashboardData();
        } catch (err: any) {
          showToast('Error removing member: ' + err.message, 'error');
        }
      }
    );
  }

  // ==========================================
  // 3. SQUAD & RECRUITMENT ACTIONS (WITH 1 SQUAD PER GAME LIMIT)
  // ==========================================
  async function handleCreateSquad(e: SubmitEvent) {
    e.preventDefault();

    if (!currentUser) {
      errorMessage = 'ကျေးဇူးပြု၍ ပထမဦးစွာ Sign in ဝင်ပါ။';
      return;
    }

    if (!squadName.trim() || !requirements.trim()) {
      errorMessage = 'အချက်အလက်အားလုံး ဖြည့်သွင်းပေးပါ။';
      return;
    }

    const existingSquadInGame = getActiveSquadForGame(gameType);
    if (existingSquadInGame) {
      errorMessage = `သင်သည် ${gameType} အတွက် "${existingSquadInGame.name}" အသင်းတွင် ပါဝင်ထားပြီးသား ဖြစ်ပါသည်။ ဂိမ်းတစ်ခုလျှင် Squad တစ်ခုသာ ဝင်ရောက်/တည်ထောင်ခွင့် ရှိပါသည်။`;
      return;
    }

    loadingAction = true;
    errorMessage = '';
    successMessage = '';

    try {
      const preservedRole: Role = isSystemAdmin ? 'admin' : 'leader';

      await supabase.from('profiles').upsert({
        id: currentUser.id,
        full_name: profileFullName || currentUser.user_metadata?.full_name || 'UIT Player',
        role: preservedRole
      });

      const { error } = await supabase.from('squads').insert({
        name: squadName.trim(),
        game_type: gameType,
        leader_id: currentUser.id,
        requirements: requirements.trim(),
        is_recruiting: true
      });

      if (error) throw error;

      if (!isSystemAdmin) {
        currentRole = 'leader';
      }

      showToast(`Squad "${squadName.trim()}" created!`, 'success');
      squadName = '';
      requirements = '';
      showCreateModal = false;

      await Promise.all([fetchSquads(), fetchDashboardData()]);
    } catch (err: any) {
      errorMessage = err.message || 'Squad တည်ထောင်ရာတွင် အမှားအယွင်း ရှိသည်။';
    } finally {
      loadingAction = false;
    }
  }

  async function handleJoinRequest(squadId: string, squadGameType: string) {
    if (!currentUser) {
      showAuthModal = true;
      return;
    }

    const existing = myJoinedSquads.find((m) => m.squad_id === squadId);
    if (existing) return;

    const activeSquadInSameGame = getActiveSquadForGame(squadGameType);
    if (activeSquadInSameGame) {
      showToast(
        `သင်သည် ${squadGameType} ဂိမ်းအတွက် "${activeSquadInSameGame.name}" အသင်းတွင် ရှိနေပြီးသားဖြစ်ပါသည် (ဂိမ်းတစ်ခုလျှင် Squad ၁ ခုသာ ဝင်နိုင်ပါသည်)။`,
        'error'
      );
      return;
    }

    const pendingInSameGame = getPendingRequestForGame(squadGameType);
    if (pendingInSameGame) {
      showToast(
        `သင်သည် ${squadGameType} အတွက် "${pendingInSameGame.squads?.name}" သို့ Request ပို့ထားပြီးသား ဖြစ်ပါသည်။`,
        'error'
      );
      return;
    }

    try {
      const { error } = await supabase.from('squad_members').insert({
        squad_id: squadId,
        user_id: currentUser.id,
        status: 'pending'
      });

      if (error) throw error;

      showToast('Join request sent to Squad Leader!', 'success');
      await fetchDashboardData();
    } catch (err: any) {
      await fetchDashboardData();
      showToast('Request ပို့ပြီးသားဖြစ်နေပါသည်: ' + err.message, 'error');
    }
  }

  async function handleMemberStatus(req: any, newStatus: 'approved' | 'rejected') {
    try {
      if (newStatus === 'approved') {
        const targetGame = req.squads?.game_type;

        const { data: applicantLed } = await supabase
          .from('squads')
          .select('name')
          .eq('leader_id', req.user_id)
          .eq('game_type', targetGame);

        if (applicantLed && applicantLed.length > 0) {
          showToast(
            `ဤကစားသမားသည် ${targetGame} တွင် "${applicantLed[0].name}" အသင်း၏ Leader ဖြစ်နေသောကြောင့် လက်ခံ၍မရပါ။`,
            'error'
          );
          return;
        }

        const { data: applicantJoined } = await supabase
          .from('squad_members')
          .select('id, squads:squad_id(name, game_type)')
          .eq('user_id', req.user_id)
          .eq('status', 'approved');

        const alreadyInSameGame = (applicantJoined || []).find(
          (m: any) => m.squads?.game_type === targetGame
        );

        if (alreadyInSameGame) {
          showToast(
            `ဤကစားသမားသည် အခြား ${targetGame} Squad တစ်ခုသို့ ဝင်ရောက်ပြီးသား ဖြစ်နေပါသည်။`,
            'error'
          );
          return;
        }
      }

      const { error } = await supabase
        .from('squad_members')
        .update({ status: newStatus })
        .eq('id', req.id);

      if (error) throw error;
      showToast(
        newStatus === 'approved' ? 'Member approved into squad!' : 'Application rejected.',
        newStatus === 'approved' ? 'success' : 'info'
      );
      await fetchDashboardData();
    } catch (err: any) {
      showToast('Error updating request: ' + err.message, 'error');
    }
  }

  // ==========================================
  // 4. SCRIM / MATCHMAKING ACTIONS
  // ==========================================
  async function handleCreateScrim(e: SubmitEvent) {
    e.preventDefault();

    if (!currentUser) {
      showAuthModal = true;
      return;
    }

    if (!challengerSquadId || !opponentSquadId || !scheduledTime) {
      showToast('အချက်အလက်များကို အပြည့်အစုံ ဖြည့်စွက်ပါ။', 'error');
      return;
    }

    if (challengerSquadId === opponentSquadId) {
      showToast('မိမိအသင်းချင်း ချိန်းပွဲဆော့၍ မရပါ။', 'error');
      return;
    }

    try {
      const selectedSquad = squads.find((s) => s.id === challengerSquadId);
      if (!selectedSquad) return;

      const { error } = await supabase.from('scrim_matches').insert({
        game_type: selectedSquad.game_type,
        challenger_squad_id: challengerSquadId,
        opponent_squad_id: opponentSquadId,
        scheduled_time: new Date(scheduledTime).toISOString(),
        status: 'pending'
      });

      if (error) throw error;

      showToast('ချိန်းပွဲ (Scrim) စိန်ခေါ်မှုကို ပို့လိုက်ပါပြီ!', 'success');
      showScrimModal = false;
      challengerSquadId = '';
      opponentSquadId = '';
      scheduledTime = '';

      await fetchScrims();
    } catch (err: any) {
      showToast('Error creating scrim: ' + err.message, 'error');
    }
  }

  async function handleScrimResponse(scrimId: string, status: 'accepted' | 'declined') {
    try {
      const { error } = await supabase
        .from('scrim_matches')
        .update({ status })
        .eq('id', scrimId);

      if (error) throw error;
      showToast(`Scrim match ${status}!`, status === 'accepted' ? 'success' : 'info');
      await fetchScrims();
    } catch (err: any) {
      showToast('Error updating scrim: ' + err.message, 'error');
    }
  }

  async function handleReportScrimWinner(scrim: any, winnerSquadId: string) {
    try {
      const loserSquadId =
        winnerSquadId === scrim.challenger_squad_id
          ? scrim.opponent_squad_id
          : scrim.challenger_squad_id;

      const { error: scrimErr } = await supabase
        .from('scrim_matches')
        .update({
          status: 'completed',
          winner_squad_id: winnerSquadId,
          challenger_confirmed: true,
          opponent_confirmed: true
        })
        .eq('id', scrim.id);

      if (scrimErr) throw scrimErr;

      const winnerSquad = squads.find((s) => s.id === winnerSquadId);
      if (winnerSquad) {
        await supabase
          .from('squads')
          .update({ wins: (winnerSquad.wins || 0) + 1 })
          .eq('id', winnerSquadId);
      }

      const loserSquad = squads.find((s) => s.id === loserSquadId);
      if (loserSquad) {
        await supabase
          .from('squads')
          .update({ losses: (loserSquad.losses || 0) + 1 })
          .eq('id', loserSquadId);
      }

      showToast('Match result recorded & standings updated!', 'success');
      await Promise.all([fetchScrims(), fetchSquads(), fetchDashboardData()]);
    } catch (err: any) {
      showToast('Error updating match result: ' + err.message, 'error');
    }
  }

  // ==========================================
  // 5. TOURNAMENT & RANDOM BRACKET ACTIONS
  // ==========================================
  async function handleCreateTournament(e: SubmitEvent) {
    e.preventDefault();
    if (!tourneyTitle.trim() || !tourneyDeadline) return;

    loadingAction = true;
    try {
      const { error } = await supabase.from('tournaments').insert({
        title: tourneyTitle.trim(),
        game_type: tourneyGameType,
        registration_deadline: new Date(tourneyDeadline).toISOString(),
        status: 'open'
      });

      if (error) throw error;

      tourneyTitle = '';
      tourneyDeadline = '';
      showTourneyModal = false;
      showToast('Official tournament launched!', 'success');
      await fetchTournaments();
    } catch (err: any) {
      showToast('Error creating tournament: ' + err.message, 'error');
    } finally {
      loadingAction = false;
    }
  }

  function handleDeleteTournament(tourneyId: string, tourneyTitleStr: string) {
    askConfirm(
      'Delete Tournament',
      `Are you sure you want to delete "${tourneyTitleStr}"?`,
      async () => {
        try {
          const { error } = await supabase.from('tournaments').delete().eq('id', tourneyId);
          if (error) throw error;
          showToast('Tournament deleted.', 'info');
          await fetchTournaments();
        } catch (err: any) {
          showToast('Error deleting tournament: ' + err.message, 'error');
        }
      }
    );
  }

  async function handleRegisterTournament(tournamentId: string, tGameType: string) {
    if (!currentUser) {
      showAuthModal = true;
      return;
    }

    const chosenSquadId = selectedSquadForTourney[tournamentId];
    if (!chosenSquadId) {
      showToast(`ပြိုင်ပွဲဝင်မည့် သင်၏ ${tGameType} Squad ကို အရင်ရွေးချယ်ပါ။`, 'error');
      return;
    }

    try {
      const { error } = await supabase.from('tournament_participants').insert({
        tournament_id: tournamentId,
        squad_id: chosenSquadId
      });

      if (error) throw error;
      showToast('ပြိုင်ပွဲအတွက် စာရင်းသွင်းခြင်း အောင်မြင်ပါသည်!', 'success');
      await fetchTournaments();
    } catch {
      showToast('ဤအသင်းသည် စာရင်းသွင်းပြီးသား ဖြစ်နေပါသည်။', 'error');
    }
  }

  async function handleGenerateRandomBracket(tournamentId: string) {
    const participants = tournamentParticipants.filter(
      (p) => p.tournament_id === tournamentId
    );

    if (participants.length < 2) {
      showToast(
        'Random ပွဲစဉ်ဇယား ဆွဲရန် အနည်းဆုံး အသင်း (၂) သင်း စာရင်းသွင်းထားရပါမည်။',
        'error'
      );
      return;
    }

    try {
      const shuffled = [...participants];
      for (let i = shuffled.length - 1; i > 0; i--) {
        const j = Math.floor(Math.random() * (i + 1));
        [shuffled[i], shuffled[j]] = [shuffled[j], shuffled[i]];
      }

      const matchesToInsert = [];
      for (let i = 0; i < shuffled.length; i += 2) {
        const squad1 = shuffled[i]?.squad_id;
        const squad2 = shuffled[i + 1]?.squad_id || null;

        matchesToInsert.push({
          tournament_id: tournamentId,
          round_number: 1,
          squad1_id: squad1,
          squad2_id: squad2,
          winner_id: squad2 ? null : squad1,
          status: squad2 ? 'scheduled' : 'completed'
        });
      }

      const { error: matchErr } = await supabase
        .from('tournament_matches')
        .insert(matchesToInsert);

      if (matchErr) throw matchErr;

      await supabase
        .from('tournaments')
        .update({ status: 'ongoing' })
        .eq('id', tournamentId);

      showToast('Random ပွဲစဉ်ဇယား (Bracket) ထုတ်ပေးပြီးပါပြီ!', 'success');
      await fetchTournaments();
    } catch (err: any) {
      showToast('Error generating bracket: ' + err.message, 'error');
    }
  }

  async function handleSetMatchWinner(matchId: string, winnerSquadId: string) {
    try {
      const { error } = await supabase
        .from('tournament_matches')
        .update({ winner_id: winnerSquadId, status: 'completed' })
        .eq('id', matchId);

      if (error) throw error;
      showToast('Match winner recorded!', 'success');
      await fetchTournaments();
    } catch (err: any) {
      showToast('Error setting match winner: ' + err.message, 'error');
    }
  }

  async function handleDeclareChampion(tournamentId: string, championSquadId: string) {
    try {
      const { error } = await supabase
        .from('tournaments')
        .update({
          status: 'completed',
          champion_squad_id: championSquadId
        })
        .eq('id', tournamentId);

      if (error) throw error;
      showToast('🏆 Official Tournament Champion declared!', 'success');
      await fetchTournaments();
    } catch (err: any) {
      showToast('Error declaring champion: ' + err.message, 'error');
    }
  }

  // ==========================================
  // 6. UI HELPERS & DERIVED FILTERS
  // ==========================================
  function openCreateModal() {
    if (!currentUser) {
      showAuthModal = true;
      return;
    }
    errorMessage = '';
    successMessage = '';
    for (const g of ['MLBB', 'PUBG', 'FOOTBALL']) {
      if (!getActiveSquadForGame(g)) {
        gameType = g;
        break;
      }
    }
    showCreateModal = true;
  }

  function closeCreateModal() {
    showCreateModal = false;
    errorMessage = '';
  }

  function openScrimModal() {
    if (!currentUser) {
      showAuthModal = true;
      return;
    }
    showScrimModal = true;
  }

  function closeScrimModal() {
    showScrimModal = false;
  }

  function openProfileModal() {
    errorMessage = '';
    successMessage = '';
    showProfileModal = true;
  }

  async function changeGameFilter(game: string) {
    selectedGameFilter = game;
    activeTab = 'squads';
    await fetchSquads();
  }

  let filteredSquads = $derived(
    squads.filter((sq) => {
      const matchesSearch =
        !squadSearchQuery.trim() ||
        sq.name?.toLowerCase().includes(squadSearchQuery.toLowerCase()) ||
        sq.requirements?.toLowerCase().includes(squadSearchQuery.toLowerCase());
      const matchesOpen = !showOpenSquadsOnly || sq.is_recruiting;
      return matchesSearch && matchesOpen;
    })
  );

  let filteredTournaments = $derived(
    tournaments.filter((t) =>
      tourneyStatusFilter === 'ALL' ? true : t.status === tourneyStatusFilter
    )
  );

  let filteredMembers = $derived(
    allProfiles.filter((p) => {
      const q = memberSearchQuery.toLowerCase().trim();
      const matchesSearch =
        !q ||
        p.full_name?.toLowerCase().includes(q) ||
        p.student_id?.toLowerCase().includes(q) ||
        p.mlbb_id?.toLowerCase().includes(q) ||
        p.pubg_id?.toLowerCase().includes(q) ||
        p.football_position?.toLowerCase().includes(q);

      const matchesGame =
        memberGameFilter === 'ALL' ||
        (memberGameFilter === 'MLBB' && p.mlbb_id) ||
        (memberGameFilter === 'PUBG' && p.pubg_id) ||
        (memberGameFilter === 'FOOTBALL' && p.football_position);

      return matchesSearch && matchesGame;
    })
  );

  let currentDmConversation = $derived(
    directMessages.filter(
      (m) =>
        currentUser &&
        ((m.sender_id === currentUser.id && m.receiver_id === selectedDmPartnerId) ||
          (m.sender_id === selectedDmPartnerId && m.receiver_id === currentUser.id))
    )
  );

  let dmPartnerList = $derived(
    allProfiles.filter((p) => {
      if (currentUser && p.id === currentUser.id) return false;
      if (!dmSearchQuery.trim()) return true;
      const q = dmSearchQuery.toLowerCase();
      return (
        p.full_name?.toLowerCase().includes(q) ||
        p.student_id?.toLowerCase().includes(q)
      );
    })
  );

  let unreadNotificationsCount = $derived(
    pendingMembers.filter((m) => m.status === 'pending').length +
      scrims.filter(
        (s) =>
          s.status === 'pending' &&
          currentUser &&
          s.opponent?.leader_id === currentUser.id
      ).length
  );
</script>

<!-- ========================================================= -->
<!-- UNIFIED DARK NEON ESPORTS DASHBOARD SHELL (SIDEBAR + MAIN) -->
<!-- ========================================================= -->
<div class="min-h-screen bg-[#06080f] text-slate-100 font-sans selection:bg-purple-500 selection:text-white flex">

  <!-- AMBIENT NEON BACKGROUND GLOWS -->
  <div class="fixed top-0 left-64 w-125 h-87.5 bg-purple-600/10 rounded-full blur-[130px] pointer-events-none z-0"></div>
  <div class="fixed bottom-0 right-10 w-105 h-80 bg-emerald-500/8 rounded-full blur-[130px] pointer-events-none z-0"></div>

  <!-- TOAST NOTIFICATION STACK -->
  <div class="fixed bottom-5 right-5 z-100 space-y-2.5 max-w-sm w-full pointer-events-none px-4">
    {#each toasts as t (t.id)}
      <div class="pointer-events-auto flex items-center justify-between px-4 py-3 rounded-2xl border backdrop-blur-xl shadow-2xl transition-all {t.type === 'success' ? 'bg-slate-900/95 border-emerald-500/50 text-emerald-300 shadow-emerald-500/10' : t.type === 'error' ? 'bg-slate-900/95 border-red-500/50 text-red-300 shadow-red-500/10' : 'bg-slate-900/95 border-purple-500/50 text-purple-300 shadow-purple-500/10'}">
        <div class="flex items-center space-x-2.5 text-xs font-bold">
          <CheckCircle2 class="w-4 h-4 shrink-0" />
          <span>{t.text}</span>
        </div>
      </div>
    {/each}
  </div>

  <!-- ========================================================= -->
  <!-- DESKTOP SIDEBAR NAVIGATION -->
  <!-- ========================================================= -->
  <aside class="hidden lg:flex flex-col w-64 bg-[#090c15]/95 border-r border-purple-500/15 sticky top-0 h-screen z-40 justify-between p-5 shrink-0">
    <div class="space-y-6">
      <!-- Brand Logo -->
      <button
        type="button"
        onclick={() => (activeTab = 'hub')}
        class="flex items-center space-x-3 text-left group cursor-pointer w-full"
      >
        <div class="w-11 h-11 rounded-2xl bg-linear-to-br from-purple-600 via-violet-600 to-fuchsia-600 flex items-center justify-center shadow-lg shadow-purple-600/30 border border-purple-400/30">
          <Trophy class="w-6 h-6 text-white" />
        </div>
        <div>
          <span class="font-black text-lg tracking-wider bg-linear-to-r from-white via-purple-200 to-purple-400 bg-clip-text text-transparent block leading-none">
            UIT ESPORTS
          </span>
          <span class="flex items-center space-x-1.5 text-[10px] tracking-widest text-emerald-400 uppercase font-bold mt-1">
            <span class="w-1.5 h-1.5 rounded-full bg-emerald-400 animate-pulse"></span>
            <span>Arena Online</span>
          </span>
        </div>
      </button>

      <!-- Navigation Links -->
      <div class="space-y-1">
        <span class="px-3 text-[10px] font-extrabold uppercase tracking-widest text-slate-500 block mb-1.5">
          Main Arena
        </span>

        <button
          type="button"
          onclick={() => (activeTab = 'hub')}
          class="w-full flex items-center space-x-3 px-3.5 py-2.5 rounded-xl text-sm font-semibold transition-all cursor-pointer {activeTab === 'hub' ? 'bg-linear-to-r from-purple-600 to-violet-600 text-white shadow-lg shadow-purple-600/25 font-bold border border-purple-400/30' : 'text-slate-400 hover:text-white hover:bg-slate-800/50'}"
        >
          <Gamepad2 class="w-4 h-4" />
          <span>Game Hub</span>
        </button>

        <button
          type="button"
          onclick={() => (activeTab = 'squads')}
          class="w-full flex items-center space-x-3 px-3.5 py-2.5 rounded-xl text-sm font-semibold transition-all cursor-pointer {activeTab === 'squads' ? 'bg-linear-to-r from-purple-600 to-violet-600 text-white shadow-lg shadow-purple-600/25 font-bold border border-purple-400/30' : 'text-slate-400 hover:text-white hover:bg-slate-800/50'}"
        >
          <Shield class="w-4 h-4" />
          <span>Squads & Recruit</span>
        </button>

        <button
          type="button"
          onclick={() => {
            activeTab = 'mysquad';
            fetchDashboardData();
          }}
          class="w-full flex items-center justify-between px-3.5 py-2.5 rounded-xl text-sm font-semibold transition-all cursor-pointer {activeTab === 'mysquad' ? 'bg-linear-to-r from-purple-600 to-violet-600 text-white shadow-lg shadow-purple-600/25 font-bold border border-purple-400/30' : 'text-slate-400 hover:text-white hover:bg-slate-800/50'}"
        >
          <div class="flex items-center space-x-3">
            <Users class="w-4 h-4" />
            <span>My Squad HQ</span>
          </div>
          {#if activeMySquadsList.length > 0}
            <span class="px-2 py-0.5 text-[10px] rounded-full bg-emerald-500/20 text-emerald-400 border border-emerald-500/30 font-extrabold">
              {activeMySquadsList.length}
            </span>
          {/if}
        </button>

        <button
          type="button"
          onclick={() => (activeTab = 'scrims')}
          class="w-full flex items-center space-x-3 px-3.5 py-2.5 rounded-xl text-sm font-semibold transition-all cursor-pointer {activeTab === 'scrims' ? 'bg-linear-to-r from-purple-600 to-violet-600 text-white shadow-lg shadow-purple-600/25 font-bold border border-purple-400/30' : 'text-slate-400 hover:text-white hover:bg-slate-800/50'}"
        >
          <Swords class="w-4 h-4" />
          <span>Scrims & Matches</span>
        </button>

        <button
          type="button"
          onclick={() => (activeTab = 'tournaments')}
          class="w-full flex items-center space-x-3 px-3.5 py-2.5 rounded-xl text-sm font-semibold transition-all cursor-pointer {activeTab === 'tournaments' ? 'bg-linear-to-r from-purple-600 to-violet-600 text-white shadow-lg shadow-purple-600/25 font-bold border border-purple-400/30' : 'text-slate-400 hover:text-white hover:bg-slate-800/50'}"
        >
          <Trophy class="w-4 h-4" />
          <span>Tournaments</span>
        </button>

        <span class="px-3 pt-3 text-[10px] font-extrabold uppercase tracking-widest text-slate-500 block mb-1.5">
          Community & Direct Chat
        </span>

        <button
          type="button"
          onclick={() => {
            activeTab = 'members';
            fetchAllMembers();
          }}
          class="w-full flex items-center space-x-3 px-3.5 py-2.5 rounded-xl text-sm font-semibold transition-all cursor-pointer {activeTab === 'members' ? 'bg-linear-to-r from-purple-600 to-violet-600 text-white shadow-lg shadow-purple-600/25 font-bold border border-purple-400/30' : 'text-slate-400 hover:text-white hover:bg-slate-800/50'}"
        >
          <Globe class="w-4 h-4" />
          <span>Active Players</span>
        </button>

        <button
          type="button"
          onclick={() => {
            activeTab = 'messages';
            fetchAllMembers();
            fetchDirectMessages();
          }}
          class="w-full flex items-center justify-between px-3.5 py-2.5 rounded-xl text-sm font-semibold transition-all cursor-pointer {activeTab === 'messages' ? 'bg-linear-to-r from-purple-600 to-violet-600 text-white shadow-lg shadow-purple-600/25 font-bold border border-purple-400/30' : 'text-slate-400 hover:text-white hover:bg-slate-800/50'}"
        >
          <div class="flex items-center space-x-3">
            <Mail class="w-4 h-4" />
            <span>Direct Messages</span>
          </div>
          <span class="px-1.5 py-0.5 text-[9px] rounded bg-purple-500/20 text-purple-300 font-bold">
            1-on-1
          </span>
        </button>

        <button
          type="button"
          onclick={() => {
            activeTab = 'allchat';
            fetchGlobalChat();
          }}
          class="w-full flex items-center justify-between px-3.5 py-2.5 rounded-xl text-sm font-semibold transition-all cursor-pointer {activeTab === 'allchat' ? 'bg-linear-to-r from-purple-600 to-violet-600 text-white shadow-lg shadow-purple-600/25 font-bold border border-purple-400/30' : 'text-slate-400 hover:text-white hover:bg-slate-800/50'}"
        >
          <div class="flex items-center space-x-3">
            <MessageSquare class="w-4 h-4" />
            <span>All Chat</span>
          </div>
          <span class="w-2 h-2 rounded-full bg-emerald-400"></span>
        </button>

        <button
          type="button"
          onclick={() => {
            activeTab = 'dashboard';
            fetchDashboardData();
          }}
          class="w-full flex items-center justify-between px-3.5 py-2.5 rounded-xl text-sm font-semibold transition-all cursor-pointer {activeTab === 'dashboard' ? 'bg-linear-to-r from-purple-600 to-violet-600 text-white shadow-lg shadow-purple-600/25 font-bold border border-purple-400/30' : 'text-slate-400 hover:text-white hover:bg-slate-800/50'}"
        >
          <div class="flex items-center space-x-3">
            <LayoutDashboard class="w-4 h-4" />
            <span>{isSystemAdmin ? 'Admin & Dashboard' : 'My Dashboard'}</span>
          </div>
          {#if unreadNotificationsCount > 0}
            <span class="px-2 py-0.5 text-[10px] rounded-full bg-purple-500 text-white font-extrabold">
              {unreadNotificationsCount}
            </span>
          {/if}
        </button>
      </div>
    </div>

    <!-- Sidebar Footer Quick CTA -->
    <div class="bg-linear-to-br from-purple-950/60 to-slate-900/90 border border-purple-500/25 rounded-2xl p-4 space-y-3">
      <div class="flex items-center space-x-2">
        <Sparkles class="w-4 h-4 text-purple-400" />
        <span class="text-xs font-extrabold text-white">Multi-Role Enabled</span>
      </div>
      <p class="text-[11px] text-slate-400 leading-relaxed">
        1 Squad per game category. Hold Leader & Member roles across games.
      </p>
      <button
        type="button"
        onclick={openCreateModal}
        class="w-full bg-linear-to-r from-purple-600 to-violet-600 hover:from-purple-500 hover:to-violet-500 text-white font-bold py-2 rounded-xl text-xs shadow-lg shadow-purple-600/25 transition cursor-pointer"
      >
        + Create Squad
      </button>
    </div>
  </aside>

  <!-- ========================================================= -->
  <!-- MAIN CONTENT WRAPPER (TOP HEADER + VIEWPORT) -->
  <!-- ========================================================= -->
  <div class="flex-1 flex flex-col min-w-0 relative z-10">

    <!-- TOP BAR -->
    <header class="h-20 border-b border-slate-800/80 bg-[#090c15]/85 backdrop-blur-xl sticky top-0 z-30 px-4 md:px-8 flex items-center justify-between gap-4">
      <div class="flex items-center space-x-3">
        <button
          type="button"
          onclick={() => (mobileMenuOpen = !mobileMenuOpen)}
          class="lg:hidden bg-slate-900 border border-slate-800 p-2 rounded-xl text-slate-300 cursor-pointer"
        >
          <Menu class="w-5 h-5" />
        </button>
        <div>
          <h2 class="text-base md:text-lg font-extrabold text-white capitalize">
            {activeTab === 'hub'
              ? 'Campus Esports Hub'
              : activeTab === 'mysquad'
                ? 'My Squad Headquarters'
                : activeTab === 'messages'
                  ? 'Direct Player Messages (1-on-1)'
                  : activeTab === 'allchat'
                    ? 'Global Community Chat'
                    : activeTab}
          </h2>
          <p class="text-[11px] text-slate-400 hidden sm:block">
            University of Information Technology • Competitive Ecosystem
          </p>
        </div>
      </div>

      <!-- Right Controls: Notifications, Multi-Role Profile Badge & Auth -->
      <div class="flex items-center space-x-3">
        {#if currentUser}
          <!-- Notification Center Bell -->
          <div class="relative">
            <button
              type="button"
              onclick={() => (showNotifications = !showNotifications)}
              class="relative bg-slate-900 hover:bg-slate-800 border border-slate-800 p-2.5 rounded-xl text-slate-300 hover:text-white transition cursor-pointer"
              title="Notifications"
            >
              <Bell class="w-4 h-4" />
              {#if unreadNotificationsCount > 0}
                <span class="absolute -top-1 -right-1 w-4 h-4 rounded-full bg-emerald-500 text-slate-950 text-[10px] font-black flex items-center justify-center">
                  {unreadNotificationsCount}
                </span>
              {/if}
            </button>

            {#if showNotifications}
              <div class="absolute right-0 mt-2 w-80 bg-[#0d121f] border border-purple-500/30 rounded-2xl shadow-2xl p-4 z-50 space-y-3">
                <div class="flex justify-between items-center border-b border-slate-800 pb-2">
                  <span class="text-xs font-extrabold uppercase tracking-wider text-purple-400">
                    Notifications ({unreadNotificationsCount})
                  </span>
                  <button
                    type="button"
                    onclick={() => (showNotifications = false)}
                    class="text-slate-400 hover:text-white text-xs cursor-pointer"
                  >
                    Close
                  </button>
                </div>
                {#if unreadNotificationsCount === 0}
                  <p class="text-xs text-slate-400 py-4 text-center">
                    No pending squad requests or scrim alerts.
                  </p>
                {:else}
                  <div class="space-y-2 max-h-60 overflow-y-auto">
                    {#each pendingMembers.filter((m) => m.status === 'pending') as req}
                      <button
                        type="button"
                        onclick={() => {
                          showNotifications = false;
                          activeTab = 'dashboard';
                        }}
                        class="w-full text-left bg-slate-900 hover:bg-slate-800 p-2.5 rounded-xl border border-slate-800 text-xs block cursor-pointer"
                      >
                        <strong class="text-white">{req.profiles?.full_name || 'Player'}</strong> requested to join <span class="text-purple-400 font-bold">{req.squads?.name}</span>
                      </button>
                    {/each}
                  </div>
                {/if}
              </div>
            {/if}
          </div>

          <!-- Profile & Multi-Role Badges -->
          <button
            type="button"
            onclick={openProfileModal}
            class="bg-slate-900/90 hover:bg-slate-800 border border-purple-500/30 px-3.5 py-2 rounded-xl flex items-center space-x-2 text-xs transition cursor-pointer shadow-sm"
          >
            <div class="w-6 h-6 rounded-lg bg-linear-to-br from-purple-500 to-violet-600 text-white font-black flex items-center justify-center text-[11px]">
              {(profileFullName || 'U')[0].toUpperCase()}
            </div>
            <span class="text-slate-200 font-bold hidden sm:inline">{profileFullName || 'Player'}</span>
            {#if profileStudentId}
              <span class="text-[10px] font-mono text-purple-300 hidden md:inline">({profileStudentId})</span>
            {/if}

            <!-- Multi-Role Badges Display -->
            <div class="flex items-center space-x-1">
              {#if isSystemAdmin}
                <span class="px-2 py-0.5 rounded-md bg-amber-500/20 text-amber-300 border border-amber-500/40 font-extrabold uppercase text-[10px]">
                  👑 ADMIN
                </span>
              {/if}
              {#if isAnySquadLeader}
                <span class="px-2 py-0.5 rounded-md bg-purple-500/20 text-purple-300 border border-purple-500/30 font-extrabold uppercase text-[10px]">
                  🛡️ LEADER
                </span>
              {/if}
              {#if isAnySquadMember || (!isSystemAdmin && !isAnySquadLeader)}
                <span class="px-2 py-0.5 rounded-md bg-emerald-500/15 text-emerald-300 border border-emerald-500/30 font-extrabold uppercase text-[10px] {isSystemAdmin || isAnySquadLeader ? 'hidden xl:inline-block' : ''}">
                  🎮 MEMBER
                </span>
              {/if}
            </div>
          </button>

          <button
            type="button"
            onclick={handleSignOut}
            class="bg-slate-900 hover:bg-red-500/20 hover:text-red-400 border border-slate-800 p-2.5 rounded-xl text-slate-400 transition cursor-pointer"
            title="Sign Out"
          >
            <LogOut class="w-4 h-4" />
          </button>
        {:else}
          <button
            type="button"
            onclick={() => {
              errorMessage = '';
              successMessage = '';
              showAuthModal = true;
            }}
            class="bg-linear-to-r from-purple-600 to-violet-600 hover:from-purple-500 hover:to-violet-500 text-white px-4 py-2.5 rounded-xl text-xs font-extrabold flex items-center space-x-1.5 transition shadow-lg shadow-purple-600/25 cursor-pointer"
          >
            <LogIn class="w-4 h-4" />
            <span>Sign In</span>
          </button>
        {/if}
      </div>
    </header>

    <!-- MOBILE NAVIGATION BAR -->
    {#if mobileMenuOpen}
      <div class="lg:hidden bg-[#090c15] border-b border-purple-500/20 px-4 py-3 grid grid-cols-2 gap-2 z-30">
        {#each [['hub', 'Game Hub'], ['squads', 'Squads'], ['mysquad', 'My Squad'], ['scrims', 'Scrims'], ['tournaments', 'Tournaments'], ['members', 'Active Players'], ['messages', 'Direct Chat'], ['allchat', 'All Chat'], ['dashboard', 'Dashboard']] as [key, label]}
          <button
            type="button"
            onclick={() => {
              activeTab = key as Tab;
              mobileMenuOpen = false;
              if (key === 'dashboard' || key === 'mysquad') fetchDashboardData();
              if (key === 'members' || key === 'messages') fetchAllMembers();
            }}
            class="px-3 py-2 rounded-xl text-xs font-bold text-left cursor-pointer {activeTab === key ? 'bg-purple-600 text-white' : 'bg-slate-900 text-slate-400'}"
          >
            {label}
          </button>
        {/each}
      </div>
    {/if}

    <!-- ========================================================= -->
    <!-- MAIN PAGE CONTENT -->
    <!-- ========================================================= -->
    <main class="max-w-7xl w-full mx-auto px-4 md:px-8 py-8 flex-1">

      <!-- ================================= -->
      <!-- 1. GAME HUB -->
      <!-- ================================= -->
      {#if activeTab === 'hub'}
        <div class="space-y-10">

          <!-- HERO BANNER WITH PRESERVED /background.jpg -->
          <div class="relative overflow-hidden rounded-3xl border border-purple-500/30 p-8 md:p-14 shadow-2xl shadow-purple-950/20">
            <img
              src="/background.jpg"
              alt="Esports Arena Background"
              class="absolute inset-0 w-full h-full object-cover object-center opacity-35 scale-105"
            />
            <div class="absolute inset-0 bg-linear-to-r from-[#06080f] via-[#06080f]/90 to-purple-950/40"></div>

            <div class="relative z-10 max-w-2xl">
              <div class="inline-flex items-center space-x-2 bg-purple-500/15 border border-purple-400/40 px-3.5 py-1 rounded-full text-purple-300 text-xs font-bold uppercase tracking-wider mb-4 backdrop-blur-md">
                <span class="w-2 h-2 rounded-full bg-emerald-400 animate-pulse"></span>
                <span>UIT Official Campus Esports Ecosystem</span>
              </div>
              <h1 class="text-4xl md:text-5xl font-black text-white tracking-tight mb-4 leading-tight drop-shadow-lg">
                Forge Your Squad. Dominate The Campus Arena.
              </h1>
              <p class="text-slate-300 text-base md:text-lg mb-8 leading-relaxed">
                UIT ကျောင်းသားများအတွက် MLBB၊ PUBG နှင့် Football Squad များဖွဲ့စည်းရန်၊ အသင်းချင်း Scrim ချိန်းဆိုရန်နှင့် တရားဝင်ပြိုင်ပွဲများတွင် ပါဝင်ယှဉ်ပြိုင်ရန်။
              </p>
              <div class="flex flex-wrap gap-4">
                <button
                  type="button"
                  onclick={() => (activeTab = 'squads')}
                  class="bg-linear-to-r from-purple-600 to-violet-600 hover:from-purple-500 hover:to-violet-500 text-white font-extrabold px-7 py-3.5 rounded-2xl shadow-xl shadow-purple-600/30 transition-all transform hover:-translate-y-0.5 flex items-center space-x-2 cursor-pointer border border-purple-400/30"
                >
                  <span>Explore Squads</span>
                  <ChevronRight class="w-5 h-5" />
                </button>
                <button
                  type="button"
                  onclick={openCreateModal}
                  class="bg-slate-900/85 hover:bg-slate-800 backdrop-blur-md text-white font-bold px-6 py-3.5 rounded-2xl border border-slate-700 transition-all cursor-pointer"
                >
                  + Create Squad
                </button>
                <button
                  type="button"
                  onclick={() => {
                    activeTab = 'members';
                    fetchAllMembers();
                  }}
                  class="bg-emerald-500/15 hover:bg-emerald-500/25 text-emerald-300 font-bold px-5 py-3.5 rounded-2xl border border-emerald-500/30 transition-all cursor-pointer"
                >
                  Find & Message Players ({allProfiles.length})
                </button>
              </div>
            </div>
          </div>

          <!-- QUICK ECOSYSTEM STATS -->
          <div class="grid grid-cols-2 md:grid-cols-4 gap-4">
            <div class="bg-[#0b0f1a] border border-slate-800/80 rounded-2xl p-5">
              <span class="text-xs text-slate-400 font-semibold block">Active Squads</span>
              <span class="text-2xl font-black text-white mt-1 block">{squads.length}</span>
            </div>
            <div class="bg-[#0b0f1a] border border-slate-800/80 rounded-2xl p-5">
              <span class="text-xs text-slate-400 font-semibold block">Registered Players</span>
              <span class="text-2xl font-black text-purple-400 mt-1 block">{allProfiles.length}</span>
            </div>
            <div class="bg-[#0b0f1a] border border-slate-800/80 rounded-2xl p-5">
              <span class="text-xs text-slate-400 font-semibold block">Scrim Battles</span>
              <span class="text-2xl font-black text-emerald-400 mt-1 block">{scrims.length}</span>
            </div>
            <div class="bg-[#0b0f1a] border border-slate-800/80 rounded-2xl p-5">
              <span class="text-xs text-slate-400 font-semibold block">Tournaments</span>
              <span class="text-2xl font-black text-amber-400 mt-1 block">{tournaments.length}</span>
            </div>
          </div>

          <!-- FEATURED GAME CATEGORIES -->
          <div class="space-y-4">
            <div class="flex justify-between items-center">
              <h2 class="text-2xl font-extrabold text-white tracking-tight">
                Featured Game Arenas
              </h2>
              <button
                type="button"
                onclick={() => (activeTab = 'squads')}
                class="text-xs text-purple-400 font-bold cursor-pointer hover:underline"
              >
                View All Squads →
              </button>
            </div>

            <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
              <!-- 1. MLBB CARD -->
              <div class="group relative overflow-hidden rounded-3xl bg-[#0b0f1a] border border-slate-800/90 hover:border-purple-500/60 transition-all duration-300 flex flex-col justify-between shadow-xl">
                <div class="relative h-44 overflow-hidden">
                  <img
                    src="/mlbb1.jpg"
                    alt="MLBB Cover"
                    class="w-full h-full object-cover group-hover:scale-110 transition-transform duration-500 opacity-65"
                  />
                  <div class="absolute inset-0 bg-linear-to-t from-[#0b0f1a] via-[#0b0f1a]/40 to-transparent"></div>

                  <div class="absolute bottom-3 left-6 flex items-center space-x-3">
                    <img
                      src="/mlbb2.jpg"
                      alt="MLBB Logo"
                      class="w-14 h-14 rounded-2xl bg-slate-950 p-1 border border-purple-500/40 shadow-lg object-cover"
                    />
                    <div>
                      <span class="text-[10px] font-extrabold uppercase tracking-wider px-2 py-0.5 rounded bg-purple-500/20 text-purple-300 border border-purple-500/30">
                        5v5 MOBA
                      </span>
                      <h3 class="text-xl font-extrabold text-white mt-0.5">MLBB</h3>
                    </div>
                  </div>
                </div>

                <div class="p-6 pt-3 space-y-5">
                  <p class="text-slate-400 text-sm leading-relaxed">
                    Mythic, Epic players များစုဝေးရာ။ 5v5 Scrims နဲ့ University Cup တွေအတွက် Squad ဖွဲ့ပါ။
                  </p>
                  <button
                    type="button"
                    onclick={() => changeGameFilter('MLBB')}
                    class="w-full bg-slate-900 hover:bg-purple-600 text-slate-200 hover:text-white py-3 rounded-xl font-bold text-sm transition-all border border-slate-800 hover:border-purple-400 cursor-pointer"
                  >
                    Browse MLBB Squads
                  </button>
                </div>
              </div>

              <!-- 2. PUBG MOBILE CARD -->
              <div class="group relative overflow-hidden rounded-3xl bg-[#0b0f1a] border border-slate-800/90 hover:border-purple-500/60 transition-all duration-300 flex flex-col justify-between shadow-xl">
                <div class="relative h-44 overflow-hidden">
                  <img
                    src="/pubg.jpg"
                    alt="PUBG Cover"
                    class="w-full h-full object-cover group-hover:scale-110 transition-transform duration-500 opacity-65"
                  />
                  <div class="absolute inset-0 bg-linear-to-t from-[#0b0f1a] via-[#0b0f1a]/40 to-transparent"></div>

                  <div class="absolute bottom-3 left-6 flex items-center space-x-3">
                    <img
                      src="/pubg.jpg"
                      alt="PUBG Logo"
                      class="w-14 h-14 rounded-2xl bg-slate-950 p-1 border border-purple-500/40 shadow-lg object-cover"
                    />
                    <div>
                      <span class="text-[10px] font-extrabold uppercase tracking-wider px-2 py-0.5 rounded bg-purple-500/20 text-purple-300 border border-purple-500/30">
                        Battle Royale
                      </span>
                      <h3 class="text-xl font-extrabold text-white mt-0.5">PUBG Mobile</h3>
                    </div>
                  </div>
                </div>

                <div class="p-6 pt-3 space-y-5">
                  <p class="text-slate-400 text-sm leading-relaxed">
                    Classic squads နဲ့ Tournament custom rooms တွေအတွက် အဖွဲ့သားရှာဖွေရန်။
                  </p>
                  <button
                    type="button"
                    onclick={() => changeGameFilter('PUBG')}
                    class="w-full bg-slate-900 hover:bg-purple-600 text-slate-200 hover:text-white py-3 rounded-xl font-bold text-sm transition-all border border-slate-800 hover:border-purple-400 cursor-pointer"
                  >
                    Browse PUBG Squads
                  </button>
                </div>
              </div>

              <!-- 3. FOOTBALL / FUTSAL CARD -->
              <div class="group relative overflow-hidden rounded-3xl bg-[#0b0f1a] border border-slate-800/90 hover:border-emerald-500/60 transition-all duration-300 flex flex-col justify-between shadow-xl">
                <div class="relative h-44 overflow-hidden">
                  <img
                    src="https://images.unsplash.com/photo-1574629810360-7efbbe195018?auto=format&fit=crop&w=800&q=80"
                    alt="Football Cover"
                    class="w-full h-full object-cover group-hover:scale-110 transition-transform duration-500 opacity-65"
                  />
                  <div class="absolute inset-0 bg-linear-to-t from-[#0b0f1a] via-[#0b0f1a]/40 to-transparent"></div>

                  <div class="absolute bottom-3 left-6 flex items-center space-x-3">
                    <img
                      src="https://img.icons8.com/color/96/football2.png"
                      alt="Football Logo"
                      class="w-14 h-14 rounded-2xl bg-slate-950 p-1.5 border border-emerald-500/40 shadow-lg object-contain"
                    />
                    <div>
                      <span class="text-[10px] font-extrabold uppercase tracking-wider px-2 py-0.5 rounded bg-emerald-500/20 text-emerald-400 border border-emerald-500/30">
                        Campus Sports
                      </span>
                      <h3 class="text-xl font-extrabold text-white mt-0.5">Football / Futsal</h3>
                    </div>
                  </div>
                </div>

                <div class="p-6 pt-3 space-y-5">
                  <p class="text-slate-400 text-sm leading-relaxed">
                    စနေ/တနင်္ဂနွေ Futsal ပွဲစဉ်များအတွက် ကျောင်းသားချင်း ချိတ်ဆက်ကစားရန်။
                  </p>
                  <button
                    type="button"
                    onclick={() => changeGameFilter('FOOTBALL')}
                    class="w-full bg-slate-900 hover:bg-emerald-500 text-slate-200 hover:text-slate-950 py-3 rounded-xl font-bold text-sm transition-all border border-slate-800 cursor-pointer"
                  >
                    Browse Futsal Squads
                  </button>
                </div>
              </div>
            </div>
          </div>
        </div>

      <!-- ================================= -->
      <!-- 2. SQUADS & RECRUITMENT -->
      <!-- ================================= -->
      {:else if activeTab === 'squads'}
        <div class="space-y-8">
          <div class="bg-[#0b0f1a] border border-slate-800/90 p-6 rounded-3xl space-y-4">
            <div class="flex flex-col md:flex-row justify-between items-start md:items-center gap-4">
              <div>
                <h2 class="text-2xl font-extrabold text-white">Squads & Recruitment Board</h2>
                <p class="text-slate-400 text-sm mt-1">
                  ဂိမ်းအမျိုးအစား တစ်ခုစီတွင် Squad (၁) ခုသာ ဝင်ရောက်ခွင့် သို့မဟုတ် တည်ထောင်ခွင့် ရှိပါသည်။
                </p>
              </div>
              <button
                type="button"
                onclick={openCreateModal}
                class="bg-linear-to-r from-purple-600 to-violet-600 hover:from-purple-500 hover:to-violet-500 text-white px-5 py-2.5 rounded-xl font-extrabold text-sm shadow-lg shadow-purple-600/25 transition-all flex items-center space-x-2 shrink-0 cursor-pointer"
              >
                <Plus class="w-4 h-4" />
                <span>Create Squad</span>
              </button>
            </div>

            <div class="grid grid-cols-1 md:grid-cols-4 gap-3 pt-2">
              <div class="md:col-span-2 relative">
                <Search class="w-4 h-4 text-slate-400 absolute left-3.5 top-3" />
                <input
                  type="text"
                  bind:value={squadSearchQuery}
                  placeholder="Search squad name or role requirements..."
                  class="w-full bg-slate-950 border border-slate-800 rounded-xl pl-10 pr-4 py-2.5 text-sm text-white outline-none focus:border-purple-500"
                />
              </div>

              <select
                bind:value={selectedGameFilter}
                onchange={fetchSquads}
                class="bg-slate-950 border border-slate-800 text-slate-200 text-sm font-semibold rounded-xl px-4 py-2.5 outline-none focus:border-purple-500 cursor-pointer"
              >
                <option value="ALL">All Games</option>
                <option value="MLBB">MLBB</option>
                <option value="PUBG">PUBG</option>
                <option value="FOOTBALL">Football</option>
              </select>

              <button
                type="button"
                onclick={() => (showOpenSquadsOnly = !showOpenSquadsOnly)}
                class="px-4 py-2.5 rounded-xl text-xs font-bold border transition cursor-pointer flex items-center justify-center space-x-2 {showOpenSquadsOnly ? 'bg-emerald-500/20 border-emerald-500/50 text-emerald-300' : 'bg-slate-950 border-slate-800 text-slate-400'}"
              >
                <span class="w-2 h-2 rounded-full {showOpenSquadsOnly ? 'bg-emerald-400' : 'bg-slate-600'}"></span>
                <span>Recruiting Only</span>
              </button>
            </div>
          </div>

          {#if loadingSquads}
            <div class="grid grid-cols-1 md:grid-cols-3 gap-6">
              {#each [1, 2, 3] as _}
                <div class="bg-[#0b0f1a] border border-slate-800/80 rounded-3xl p-6 h-64 animate-pulse space-y-4">
                  <div class="h-6 w-24 bg-slate-800 rounded-full"></div>
                  <div class="h-8 w-48 bg-slate-800 rounded-xl"></div>
                  <div class="h-20 bg-slate-900 rounded-2xl"></div>
                </div>
              {/each}
            </div>
          {:else if filteredSquads.length === 0}
            <div class="bg-[#0b0f1a] border border-slate-800/80 rounded-3xl p-16 text-center space-y-4">
              <AlertCircle class="w-12 h-12 text-purple-400 mx-auto opacity-80" />
              <h3 class="text-lg font-bold text-white">No Matching Squads Found</h3>
              <button
                type="button"
                onclick={openCreateModal}
                class="bg-purple-600 text-white font-bold px-6 py-2.5 rounded-xl text-sm cursor-pointer"
              >
                Create Squad Now
              </button>
            </div>
          {:else}
            <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-6">
              {#each filteredSquads as squad (squad.id)}
                {@const myRequest = myJoinedSquads.find((m) => m.squad_id === squad.id)}
                {@const activeSameGameSquad = getActiveSquadForGame(squad.game_type)}
                {@const pendingSameGameReq = getPendingRequestForGame(squad.game_type)}
                {@const myPendingCount = pendingMembers.filter(
                  (m) => m.squad_id === squad.id && m.status === 'pending'
                ).length}

                <div class="bg-[#0b0f1a] border border-slate-800/90 rounded-3xl p-6 flex flex-col justify-between hover:border-purple-500/50 transition-all shadow-lg">
                  <div class="space-y-4">
                    <div class="flex justify-between items-start">
                      <div>
                        <div class="flex items-center space-x-2 mb-2">
                          <span class="inline-block px-3 py-1 rounded-full text-xs font-bold bg-purple-500/15 text-purple-300 border border-purple-500/30">
                            {squad.game_type}
                          </span>
                          <span class="inline-flex items-center space-x-1 text-[10px] font-extrabold uppercase px-2.5 py-0.5 rounded-full {squad.is_recruiting ? 'bg-emerald-500/15 text-emerald-400 border border-emerald-500/30' : 'bg-slate-800 text-slate-400'}">
                            <span class="w-1.5 h-1.5 rounded-full {squad.is_recruiting ? 'bg-emerald-400' : 'bg-slate-500'}"></span>
                            <span>{squad.is_recruiting ? 'Recruiting' : 'Closed'}</span>
                          </span>
                        </div>
                        <h3 class="text-xl font-extrabold text-white">{squad.name}</h3>
                      </div>
                      <div class="text-right bg-slate-950 px-3 py-1.5 rounded-xl border border-slate-800">
                        <span class="text-[10px] text-slate-400 uppercase block">Record</span>
                        <span class="text-sm font-black text-emerald-400">
                          {squad.wins ?? 0}W - {squad.losses ?? 0}L
                        </span>
                      </div>
                    </div>

                    <div class="bg-slate-950/70 rounded-2xl p-4 border border-slate-800/70 space-y-1.5">
                      <span class="text-[10px] text-slate-400 uppercase tracking-wider font-extrabold block">
                        Requirements & Roles Needed:
                      </span>
                      <p class="text-sm text-slate-300 leading-relaxed">
                        {squad.requirements || 'No requirements specified.'}
                      </p>
                    </div>
                  </div>

                  <div class="pt-5 mt-5 border-t border-slate-800/80 flex items-center justify-between gap-2">
                    <div class="text-xs text-slate-400 truncate">
                      Leader:
                      <strong class="text-slate-200">
                        {squad.profiles?.full_name || 'UIT Player'}
                      </strong>
                      {#if squad.profiles?.student_id}
                        <span class="text-[10px] font-mono text-purple-400 ml-1">({squad.profiles.student_id})</span>
                      {/if}
                    </div>

                    <div class="flex items-center space-x-1.5 shrink-0">
                      {#if currentUser && squad.leader_id !== currentUser.id}
                        <button
                          type="button"
                          onclick={() => startDirectChatWith(squad.leader_id)}
                          class="bg-slate-900 hover:bg-slate-800 text-slate-300 border border-slate-700 p-2 rounded-xl text-xs cursor-pointer"
                          title="Direct Message Squad Leader"
                        >
                          <Mail class="w-3.5 h-3.5" />
                        </button>
                      {/if}

                      {#if squad.leader_id === currentUser?.id}
                        <button
                          type="button"
                          onclick={() => {
                            selectedMySquadId = squad.id;
                            activeTab = 'mysquad';
                            fetchMySquadDetails(squad.id);
                          }}
                          class="bg-purple-500/20 hover:bg-purple-600 text-purple-300 hover:text-white border border-purple-500/40 px-3.5 py-2 rounded-xl text-xs font-bold transition cursor-pointer"
                        >
                          Manage HQ {myPendingCount > 0 ? `(${myPendingCount})` : ''}
                        </button>
                      {:else}
                        {#if isAnySquadLeader || isSystemAdmin}
                          <button
                            type="button"
                            onclick={() => {
                              challengerSquadId = mySquads[0]?.id || '';
                              opponentSquadId = squad.id;
                              openScrimModal();
                            }}
                            class="bg-purple-500/15 hover:bg-purple-600 text-purple-300 hover:text-white border border-purple-500/30 px-3 py-2 rounded-xl text-xs font-bold transition cursor-pointer flex items-center space-x-1"
                          >
                            <Swords class="w-3.5 h-3.5" />
                            <span>Scrim</span>
                          </button>
                        {/if}

                        {#if myRequest?.status === 'approved'}
                          <button
                            type="button"
                            onclick={() => {
                              selectedMySquadId = squad.id;
                              activeTab = 'mysquad';
                              fetchMySquadDetails(squad.id);
                            }}
                            class="bg-emerald-500/20 hover:bg-emerald-500 text-emerald-400 hover:text-slate-950 border border-emerald-500/40 px-3.5 py-2 rounded-xl text-xs font-bold transition cursor-pointer"
                          >
                            Joined ✓
                          </button>
                        {:else if activeSameGameSquad}
                          <button
                            type="button"
                            disabled
                            class="px-3 py-2 rounded-xl text-xs font-bold bg-slate-900 text-slate-500 border border-slate-800 cursor-not-allowed"
                          >
                            In {squad.game_type} Squad
                          </button>
                        {:else if pendingSameGameReq && pendingSameGameReq.squad_id !== squad.id}
                          <button
                            type="button"
                            disabled
                            class="px-3 py-2 rounded-xl text-xs font-bold bg-slate-900 text-slate-500 border border-slate-800 cursor-not-allowed"
                          >
                            Pending Other
                          </button>
                        {:else}
                          <button
                            type="button"
                            onclick={() => handleJoinRequest(squad.id, squad.game_type)}
                            disabled={!!myRequest || !squad.is_recruiting}
                            class="px-3.5 py-2 rounded-xl text-xs font-bold transition-all border cursor-pointer disabled:cursor-not-allowed {myRequest?.status === 'pending' ? 'bg-amber-500/20 text-amber-300 border-amber-500/40' : 'bg-slate-900 hover:bg-purple-600 text-slate-200 hover:text-white border-slate-700 disabled:opacity-40'}"
                          >
                            {#if myRequest?.status === 'pending'}
                              Requested
                            {:else if myRequest?.status === 'rejected'}
                              Rejected
                            {:else if !squad.is_recruiting}
                              Closed
                            {:else}
                              Join
                            {/if}
                          </button>
                        {/if}
                      {/if}
                    </div>
                  </div>
                </div>
              {/each}
            </div>
          {/if}
        </div>

      <!-- ================================= -->
      <!-- 2.5 MY SQUAD HEADQUARTERS -->
      <!-- ================================= -->
      {:else if activeTab === 'mysquad'}
        {@const currentSquadObj = activeMySquadsList.find((s) => s.id === selectedMySquadId)}
        {@const isLeaderOfThisSquad =
          currentUser && currentSquadObj && currentSquadObj.leader_id === currentUser.id}

        <div class="space-y-8">
          <div class="flex flex-col md:flex-row justify-between items-start md:items-center gap-4 bg-[#0b0f1a] border border-slate-800/90 p-6 rounded-3xl">
            <div>
              <h2 class="text-2xl font-extrabold text-white flex items-center space-x-2.5">
                <Users class="w-6 h-6 text-purple-400" />
                <span>My Squad Headquarters</span>
              </h2>
              <p class="text-slate-400 text-sm mt-1">
                Switch between squads you lead and squads you joined as a member.
              </p>
            </div>

            {#if activeMySquadsList.length > 1}
              <div class="flex items-center space-x-2">
                <span class="text-xs text-slate-400 font-bold">Switch Squad:</span>
                <select
                  bind:value={selectedMySquadId}
                  onchange={() => fetchMySquadDetails(selectedMySquadId)}
                  class="bg-slate-950 border border-purple-500/40 text-purple-300 font-bold text-sm rounded-xl px-4 py-2.5 outline-none cursor-pointer"
                >
                  {#each activeMySquadsList as sq (sq.id)}
                    <option value={sq.id}>
                      {sq.leader_id === currentUser?.id ? '👑 [Leader]' : '🎮 [Member]'} {sq.name} ({sq.game_type})
                    </option>
                  {/each}
                </select>
              </div>
            {/if}
          </div>

          {#if !currentUser}
            <div class="bg-[#0b0f1a] border border-slate-800 rounded-3xl p-16 text-center space-y-4">
              <AlertCircle class="w-12 h-12 text-purple-400 mx-auto" />
              <h3 class="text-lg font-bold text-white">Sign In Required</h3>
              <button
                type="button"
                onclick={() => (showAuthModal = true)}
                class="bg-purple-600 text-white font-bold px-6 py-2.5 rounded-xl text-sm cursor-pointer"
              >
                Sign In Now
              </button>
            </div>
          {:else if activeMySquadsList.length === 0}
            <div class="bg-[#0b0f1a] border border-slate-800 rounded-3xl p-16 text-center space-y-4">
              <Users class="w-12 h-12 text-purple-400 mx-auto opacity-80" />
              <h3 class="text-lg font-bold text-white">You Are Not In A Squad Yet</h3>
              <div class="flex justify-center gap-3 pt-2">
                <button
                  type="button"
                  onclick={() => (activeTab = 'squads')}
                  class="bg-slate-900 hover:bg-slate-800 text-white font-bold px-5 py-2.5 rounded-xl text-xs cursor-pointer border border-slate-700"
                >
                  Browse Squads
                </button>
                <button
                  type="button"
                  onclick={openCreateModal}
                  class="bg-purple-600 text-white font-bold px-5 py-2.5 rounded-xl text-xs cursor-pointer"
                >
                  + Create Squad
                </button>
              </div>
            </div>
          {:else if currentSquadObj}
            <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">

              <!-- LEFT COLUMN: SQUAD INFO, CONTEXT ROLE CONTROLS & ROSTER -->
              <div class="bg-[#0b0f1a] border border-slate-800/90 rounded-3xl p-6 flex flex-col justify-between space-y-6">
                <div class="space-y-6">
                  <div class="bg-slate-950/90 border border-slate-800 p-5 rounded-2xl space-y-3">
                    <div class="flex justify-between items-start">
                      <div>
                        <div class="flex items-center space-x-2">
                          <span class="px-2.5 py-0.5 rounded-full text-[10px] font-bold bg-purple-500/15 text-purple-300 border border-purple-500/30">
                            {currentSquadObj.game_type}
                          </span>
                          {#if isLeaderOfThisSquad}
                            <span class="px-2 py-0.5 rounded-full text-[10px] font-extrabold uppercase bg-purple-500/20 text-purple-300 border border-purple-500/40">
                              👑 Your Role: Leader
                            </span>
                          {:else}
                            <span class="px-2 py-0.5 rounded-full text-[10px] font-extrabold uppercase bg-emerald-500/20 text-emerald-300 border border-emerald-500/40">
                              🎮 Your Role: Member
                            </span>
                          {/if}
                        </div>
                        <h3 class="text-2xl font-black text-white mt-1.5">{currentSquadObj.name}</h3>
                      </div>
                      <span class="text-sm font-extrabold text-emerald-400 bg-slate-900 px-3 py-1 rounded-xl border border-slate-800">
                        {currentSquadObj.wins ?? 0}W - {currentSquadObj.losses ?? 0}L
                      </span>
                    </div>

                    {#if editingSquadRequirements}
                      <div class="space-y-2 pt-1">
                        <textarea
                          bind:value={updatedRequirementsText}
                          rows="2"
                          class="w-full bg-slate-900 border border-purple-500/50 rounded-xl p-2.5 text-xs text-white outline-none"
                        ></textarea>
                        <div class="flex space-x-2">
                          <button
                            type="button"
                            onclick={() => handleSaveSquadRequirements(currentSquadObj.id)}
                            class="bg-purple-600 text-white px-3 py-1 rounded-lg text-xs font-bold cursor-pointer"
                          >
                            Save
                          </button>
                          <button
                            type="button"
                            onclick={() => (editingSquadRequirements = false)}
                            class="bg-slate-800 text-slate-300 px-3 py-1 rounded-lg text-xs cursor-pointer"
                          >
                            Cancel
                          </button>
                        </div>
                      </div>
                    {:else}
                      <p class="text-xs text-slate-400 leading-relaxed">
                        {currentSquadObj.requirements}
                      </p>
                    {/if}

                    {#if isLeaderOfThisSquad}
                      <div class="pt-3 border-t border-slate-800/80 flex flex-wrap gap-2">
                        <button
                          type="button"
                          onclick={() => handleToggleRecruitment(currentSquadObj)}
                          class="flex-1 bg-slate-900 hover:bg-slate-800 border border-slate-700 px-3 py-1.5 rounded-xl text-[11px] font-bold flex items-center justify-center space-x-1.5 cursor-pointer {currentSquadObj.is_recruiting ? 'text-emerald-400' : 'text-amber-400'}"
                        >
                          {#if currentSquadObj.is_recruiting}
                            <Unlock class="w-3.5 h-3.5" />
                            <span>Recruiting: Open</span>
                          {:else}
                            <Lock class="w-3.5 h-3.5" />
                            <span>Recruiting: Closed</span>
                          {/if}
                        </button>

                        <button
                          type="button"
                          onclick={() => {
                            updatedRequirementsText = currentSquadObj.requirements || '';
                            editingSquadRequirements = true;
                          }}
                          class="bg-slate-900 hover:bg-slate-800 border border-slate-700 px-2.5 py-1.5 rounded-xl text-[11px] text-slate-300 font-bold flex items-center space-x-1 cursor-pointer"
                        >
                          <Edit3 class="w-3.5 h-3.5" />
                          <span>Edit</span>
                        </button>

                        <button
                          type="button"
                          onclick={() => handleDisbandSquad(currentSquadObj.id, currentSquadObj.name)}
                          class="bg-red-500/10 hover:bg-red-500 text-red-400 hover:text-white border border-red-500/30 px-2.5 py-1.5 rounded-xl text-[11px] font-bold flex items-center cursor-pointer"
                          title="Disband Squad"
                        >
                          <Trash2 class="w-3.5 h-3.5" />
                        </button>
                      </div>
                    {/if}
                  </div>

                  <!-- Squad Roster -->
                  <div class="space-y-3">
                    <h4 class="text-xs font-extrabold uppercase tracking-wider text-slate-400">
                      Squad Roster ({squadRoster.length + 1} Players)
                    </h4>

                    <div class="space-y-2.5">
                      <div class="bg-purple-500/10 border border-purple-500/30 px-4 py-3 rounded-2xl flex items-center justify-between">
                        <div class="flex items-center space-x-3">
                          <div class="w-8 h-8 rounded-xl bg-purple-600 text-white font-black flex items-center justify-center text-xs">
                            👑
                          </div>
                          <div>
                            <span class="text-sm font-bold text-white block">
                              {currentSquadObj.profiles?.full_name || profileFullName || 'Squad Leader'}
                            </span>
                            <span class="text-[10px] uppercase tracking-wider font-extrabold text-purple-300">
                              Squad Leader
                            </span>
                          </div>
                        </div>
                      </div>

                      {#each squadRoster as member (member.id)}
                        <div class="bg-slate-950/70 border border-slate-800 px-4 py-3 rounded-2xl flex items-center justify-between">
                          <div class="flex items-center space-x-3">
                            <div class="w-8 h-8 rounded-xl bg-slate-800 text-slate-300 font-bold flex items-center justify-center text-xs">
                              🎮
                            </div>
                            <div>
                              <span class="text-sm font-bold text-slate-200 block">
                                {member.profiles?.full_name || 'UIT Member'}
                              </span>
                              <span class="text-[10px] text-slate-400 block">
                                {getUniquePlayerTag(member.profiles)}
                                {#if member.profiles?.mlbb_id} • MLBB: {member.profiles.mlbb_id}{/if}
                                {#if member.profiles?.pubg_id} • PUBG: {member.profiles.pubg_id}{/if}
                              </span>
                            </div>
                          </div>

                          <div class="flex items-center space-x-1.5">
                            {#if member.user_id !== currentUser.id}
                              <button
                                type="button"
                                onclick={() => startDirectChatWith(member.user_id)}
                                class="text-[11px] bg-slate-800 hover:bg-purple-600 text-slate-300 hover:text-white px-2 py-1 rounded-lg font-bold transition cursor-pointer"
                                title="Message Player"
                              >
                                💬
                              </button>
                            {/if}
                            {#if isLeaderOfThisSquad}
                              <button
                                type="button"
                                onclick={() =>
                                  handleRemoveMember(
                                    member.id,
                                    member.profiles?.full_name || 'Member'
                                  )}
                                class="text-[11px] bg-red-500/10 hover:bg-red-500 text-red-400 hover:text-white px-2.5 py-1 rounded-lg font-bold transition cursor-pointer"
                              >
                                Kick
                              </button>
                            {/if}
                          </div>
                        </div>
                      {/each}
                    </div>
                  </div>
                </div>

                <!-- Context-Specific Bottom Action: Leader vs Member -->
                <div class="pt-4 border-t border-slate-800">
                  {#if isLeaderOfThisSquad}
                    <button
                      type="button"
                      onclick={() => {
                        activeTab = 'dashboard';
                        fetchDashboardData();
                      }}
                      class="w-full bg-slate-900 hover:bg-purple-600 text-slate-200 hover:text-white font-bold py-3 rounded-xl text-xs transition cursor-pointer border border-slate-800"
                    >
                      Review Applications in Dashboard →
                    </button>
                  {:else}
                    <button
                      type="button"
                      onclick={() => handleLeaveSquad(currentSquadObj.id, currentSquadObj.name)}
                      class="w-full bg-red-500/15 hover:bg-red-500 text-red-400 hover:text-white border border-red-500/30 font-extrabold py-3 rounded-xl text-xs transition cursor-pointer flex items-center justify-center space-x-2"
                    >
                      <DoorOpen class="w-4 h-4" />
                      <span>Leave Squad (အသင်းမှ ထွက်မည်)</span>
                    </button>
                  {/if}
                </div>
              </div>

              <!-- RIGHT COLUMN (2 COLS): REALTIME SQUAD CHAT -->
              <div class="lg:col-span-2 bg-[#0b0f1a] border border-slate-800/90 rounded-3xl p-6 flex flex-col justify-between h-140">
                <div class="flex items-center justify-between border-b border-slate-800 pb-4">
                  <div class="flex items-center space-x-2.5">
                    <MessageSquare class="w-5 h-5 text-purple-400" />
                    <div>
                      <h4 class="text-base font-extrabold text-white flex items-center space-x-2">
                        <span>{currentSquadObj.name} — Squad Comms</span>
                        <span class="w-2 h-2 rounded-full bg-emerald-400 animate-pulse"></span>
                      </h4>
                      <span class="text-[11px] text-slate-400">Real-time team strategy & scrim coordination</span>
                    </div>
                  </div>
                </div>

                <div class="flex-1 overflow-y-auto py-4 space-y-3 pr-1">
                  {#if loadingMySquadRoom}
                    <div class="text-center py-20 text-slate-400 text-xs">Loading squad messages...</div>
                  {:else if squadMessages.length === 0}
                    <div class="text-center py-20 space-y-2">
                      <MessageSquare class="w-10 h-10 text-slate-700 mx-auto" />
                      <p class="text-sm font-bold text-slate-400">No messages in squad chat yet</p>
                      <p class="text-xs text-slate-500">Say hello to your teammates below!</p>
                    </div>
                  {:else}
                    {#each squadMessages as msg (msg.id)}
                      {@const isMe = msg.user_id === currentUser.id}
                      <div class="flex flex-col {isMe ? 'items-end' : 'items-start'}">
                        <span class="text-[10px] text-slate-400 mb-1 px-1">
                          {msg.profiles?.full_name || 'Member'}
                          {#if msg.profiles?.student_id}({msg.profiles.student_id}){/if} • {new Date(msg.created_at).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })}
                        </span>
                        <div class="max-w-[78%] px-4 py-2.5 rounded-2xl text-sm leading-relaxed {isMe ? 'bg-linear-to-r from-purple-600 to-violet-600 text-white font-semibold rounded-br-xs shadow-lg shadow-purple-600/20' : 'bg-slate-950 border border-slate-800 text-slate-200 rounded-bl-xs'}">
                          {msg.content}
                        </div>
                      </div>
                    {/each}
                  {/if}
                </div>

                <form onsubmit={handleSendChatMessage} class="pt-3 border-t border-slate-800 space-y-2">
                  <div class="flex items-center space-x-1.5">
                    {#each quickEmojis as emoji}
                      <button
                        type="button"
                        onclick={() => (newChatMessage = `${newChatMessage} ${emoji}`.trim())}
                        class="bg-slate-950 hover:bg-slate-800 border border-slate-800 px-2.5 py-1 rounded-lg text-xs cursor-pointer"
                      >
                        {emoji}
                      </button>
                    {/each}
                  </div>
                  <div class="flex items-center space-x-2.5">
                    <input
                      type="text"
                      bind:value={newChatMessage}
                      placeholder="Message your squad..."
                      maxlength="400"
                      class="flex-1 bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-sm text-white outline-none focus:border-purple-500"
                    />
                    <button
                      type="submit"
                      class="bg-linear-to-r from-purple-600 to-violet-600 hover:from-purple-500 hover:to-violet-500 text-white font-extrabold px-5 py-3 rounded-xl transition cursor-pointer flex items-center space-x-1.5 shrink-0"
                    >
                      <Send class="w-4 h-4" />
                      <span>Send</span>
                    </button>
                  </div>
                </form>
              </div>

            </div>
          {/if}
        </div>

      <!-- ================================= -->
      <!-- 3. SCRIMS & MATCHMAKING -->
      <!-- ================================= -->
      {:else if activeTab === 'scrims'}
        <div class="space-y-8">
          <div class="flex flex-col md:flex-row justify-between items-start md:items-center gap-4 bg-[#0b0f1a] border border-slate-800/90 p-6 rounded-3xl">
            <div>
              <h2 class="text-2xl font-extrabold text-white">Squad Scrims & Challenge Matches</h2>
              <p class="text-slate-400 text-sm mt-1">
                Organize 5v5 practice scrims, accept challenges, and record official win/loss standings.
              </p>
            </div>

            {#if isAnySquadLeader || isSystemAdmin}
              <button
                type="button"
                onclick={openScrimModal}
                class="bg-linear-to-r from-purple-600 to-violet-600 hover:from-purple-500 hover:to-violet-500 text-white px-5 py-2.5 rounded-xl font-extrabold text-sm shadow-lg shadow-purple-600/25 transition-all flex items-center space-x-2 shrink-0 cursor-pointer"
              >
                <Swords class="w-4 h-4" />
                <span>Challenge Scrim</span>
              </button>
            {/if}
          </div>

          {#if loadingScrims}
            <div class="text-center py-20 text-slate-400">Loading scrim matches...</div>
          {:else if scrims.length === 0}
            <div class="bg-[#0b0f1a] border border-slate-800/80 rounded-3xl p-16 text-center space-y-4">
              <Swords class="w-12 h-12 text-purple-400 mx-auto opacity-80" />
              <h3 class="text-lg font-bold text-white">No Active Scrim Challenges</h3>
            </div>
          {:else}
            <div class="grid grid-cols-1 md:grid-cols-2 gap-6">
              {#each scrims as scrim (scrim.id)}
                {@const isInvolvedLeader =
                  isSystemAdmin ||
                  (currentUser &&
                    (scrim.challenger?.leader_id === currentUser.id ||
                      scrim.opponent?.leader_id === currentUser.id))}
                {@const isOpponentLeader =
                  isSystemAdmin ||
                  (currentUser && scrim.opponent?.leader_id === currentUser.id)}

                <div class="bg-[#0b0f1a] border border-slate-800/90 rounded-3xl p-6 space-y-4 flex flex-col justify-between">
                  <div class="space-y-4">
                    <div class="flex justify-between items-center">
                      <span class="px-3 py-1 rounded-full text-xs font-bold bg-purple-500/15 text-purple-300 border border-purple-500/30">
                        {scrim.game_type}
                      </span>
                      <span class="text-xs uppercase font-bold px-2.5 py-1 rounded-lg {scrim.status === 'completed' ? 'bg-emerald-500/15 text-emerald-400' : scrim.status === 'accepted' ? 'bg-blue-500/15 text-blue-400' : 'bg-amber-500/15 text-amber-300'}">
                        {scrim.status}
                      </span>
                    </div>

                    <div class="flex justify-between items-center text-lg font-bold text-white py-4 bg-slate-950/80 px-5 rounded-2xl border border-slate-800">
                      <div class="truncate max-w-[42%]">
                        <span>{scrim.challenger?.name || 'Squad A'}</span>
                        {#if scrim.winner_squad_id === scrim.challenger_squad_id}
                          <span class="block text-[11px] text-emerald-400 font-extrabold">🏆 WINNER</span>
                        {/if}
                      </div>
                      <span class="text-purple-400 text-sm italic font-black px-2">VS</span>
                      <div class="truncate max-w-[42%] text-right">
                        <span>{scrim.opponent?.name || 'Squad B'}</span>
                        {#if scrim.winner_squad_id === scrim.opponent_squad_id}
                          <span class="block text-[11px] text-emerald-400 font-extrabold">🏆 WINNER</span>
                        {/if}
                      </div>
                    </div>

                    <p class="text-xs text-slate-400">
                      Scheduled Time:
                      <strong class="text-slate-200">{new Date(scrim.scheduled_time).toLocaleString()}</strong>
                    </p>
                  </div>

                  {#if isInvolvedLeader}
                    <div class="pt-4 border-t border-slate-800/80">
                      {#if scrim.status === 'pending' && isOpponentLeader}
                        <div class="flex items-center justify-between gap-2">
                          <span class="text-xs text-slate-400">Opponent Leader Approval:</span>
                          <div class="flex space-x-2">
                            <button
                              type="button"
                              onclick={() => handleScrimResponse(scrim.id, 'accepted')}
                              class="bg-emerald-500/20 hover:bg-emerald-500 text-emerald-400 hover:text-slate-950 px-3 py-1.5 rounded-xl text-xs font-bold transition cursor-pointer"
                            >
                              Accept Match
                            </button>
                            <button
                              type="button"
                              onclick={() => handleScrimResponse(scrim.id, 'declined')}
                              class="bg-red-500/20 hover:bg-red-500 text-red-400 hover:text-white px-3 py-1.5 rounded-xl text-xs font-bold transition cursor-pointer"
                            >
                              Decline
                            </button>
                          </div>
                        </div>
                      {:else if scrim.status === 'accepted'}
                        <div class="space-y-2">
                          <span class="text-xs text-purple-300 font-semibold block">
                            Report Match Winner (Updates Squad Standings):
                          </span>
                          <div class="grid grid-cols-2 gap-2">
                            <button
                              type="button"
                              onclick={() => handleReportScrimWinner(scrim, scrim.challenger_squad_id)}
                              class="bg-slate-900 hover:bg-emerald-500 hover:text-slate-950 text-slate-200 py-2 px-3 rounded-xl text-xs font-bold transition cursor-pointer truncate border border-slate-800"
                            >
                              🏆 {scrim.challenger?.name} Won
                            </button>
                            <button
                              type="button"
                              onclick={() => handleReportScrimWinner(scrim, scrim.opponent_squad_id)}
                              class="bg-slate-900 hover:bg-emerald-500 hover:text-slate-950 text-slate-200 py-2 px-3 rounded-xl text-xs font-bold transition cursor-pointer truncate border border-slate-800"
                            >
                              🏆 {scrim.opponent?.name} Won
                            </button>
                          </div>
                        </div>
                      {/if}
                    </div>
                  {/if}
                </div>
              {/each}
            </div>
          {/if}
        </div>

      <!-- ================================= -->
      <!-- 4. TOURNAMENTS & RANDOM BRACKETS -->
      <!-- ================================= -->
      {:else if activeTab === 'tournaments'}
        <div class="space-y-8">
          <div class="bg-[#0b0f1a] border border-slate-800/90 p-6 rounded-3xl space-y-4">
            <div class="flex flex-col sm:flex-row justify-between items-start sm:items-center gap-4">
              <div>
                <h2 class="text-2xl font-extrabold text-white">UIT Official Tournaments</h2>
                <p class="text-slate-400 text-sm mt-1">
                  Campus championship brackets, team registrations, and live match results.
                </p>
              </div>

              {#if isSystemAdmin}
                <button
                  type="button"
                  onclick={() => (showTourneyModal = true)}
                  class="bg-linear-to-r from-purple-600 to-violet-600 text-white px-5 py-2.5 rounded-xl font-extrabold text-sm cursor-pointer flex items-center space-x-2 shrink-0"
                >
                  <Plus class="w-4 h-4" />
                  <span>Create Tournament</span>
                </button>
              {/if}
            </div>

            <div class="flex flex-wrap gap-2">
              {#each [['ALL', 'All Events'], ['open', 'Registration Open'], ['ongoing', 'Live Bracket'], ['completed', 'Completed']] as [st, label]}
                <button
                  type="button"
                  onclick={() => (tourneyStatusFilter = st as any)}
                  class="px-3.5 py-1.5 rounded-xl text-xs font-bold transition cursor-pointer {tourneyStatusFilter === st ? 'bg-purple-600 text-white' : 'bg-slate-950 text-slate-400 border border-slate-800'}"
                >
                  {label}
                </button>
              {/each}
            </div>
          </div>

          {#if loadingTournaments}
            <div class="text-center py-20 text-slate-400">Loading tournaments...</div>
          {:else if filteredTournaments.length === 0}
            <div class="bg-[#0b0f1a] border border-slate-800/80 rounded-3xl p-16 text-center space-y-4">
              <Calendar class="w-12 h-12 text-purple-400 mx-auto opacity-80" />
              <h3 class="text-lg font-bold text-white">No Tournaments Found</h3>
            </div>
          {:else}
            <div class="space-y-8">
              {#each filteredTournaments as tourney (tourney.id)}
                {@const regSquads = tournamentParticipants.filter((p) => p.tournament_id === tourney.id)}
                {@const tMatches = tournamentMatches.filter((m) => m.tournament_id === tourney.id)}
                {@const myEligibleSquads = mySquads.filter((s) => s.game_type === tourney.game_type)}

                <div class="bg-[#0b0f1a] border border-slate-800/90 rounded-3xl p-7 space-y-6">
                  <div class="flex flex-col md:flex-row justify-between items-start md:items-center gap-4 border-b border-slate-800 pb-6">
                    <div class="space-y-2">
                      <div class="flex items-center space-x-2">
                        <span class="px-3 py-1 rounded-full text-xs font-bold bg-purple-500/15 text-purple-300 border border-purple-500/30">
                          {tourney.game_type}
                        </span>
                        <span class="text-xs uppercase font-bold px-2.5 py-1 rounded-lg bg-emerald-500/15 text-emerald-400">
                          {tourney.status}
                        </span>
                        {#if isSystemAdmin}
                          <button
                            type="button"
                            onclick={() => handleDeleteTournament(tourney.id, tourney.title)}
                            class="text-xs bg-red-500/15 hover:bg-red-500 text-red-400 hover:text-white px-2.5 py-1 rounded-lg font-bold cursor-pointer"
                          >
                            Delete
                          </button>
                        {/if}
                      </div>
                      <h3 class="text-2xl font-extrabold text-white">{tourney.title}</h3>
                      <p class="text-xs text-slate-400">
                        Registration Deadline:
                        <strong class="text-slate-200">
                          {tourney.registration_deadline
                            ? new Date(tourney.registration_deadline).toLocaleString()
                            : 'Not specified'}
                        </strong>
                      </p>
                    </div>

                    {#if tourney.status === 'completed' && tourney.champion}
                      <div class="bg-linear-to-r from-amber-500/20 to-purple-600/20 border border-amber-500/50 px-6 py-4 rounded-2xl flex items-center space-x-3">
                        <Crown class="w-8 h-8 text-amber-400 shrink-0" />
                        <div>
                          <span class="text-[10px] uppercase tracking-widest text-amber-300 font-bold block">
                            Official Tournament Champion
                          </span>
                          <span class="text-xl font-black text-white">{tourney.champion.name} 🏆</span>
                        </div>
                      </div>
                    {/if}
                  </div>

                  <div class="grid grid-cols-1 lg:grid-cols-3 gap-6">
                    <div class="bg-slate-950/70 border border-slate-800/80 rounded-2xl p-5 space-y-4">
                      <h4 class="text-sm font-bold text-white uppercase tracking-wider">
                        Registered Teams ({regSquads.length})
                      </h4>

                      {#if regSquads.length === 0}
                        <p class="text-xs text-slate-400">စာရင်းသွင်းထားသော အသင်း မရှိသေးပါ။</p>
                      {:else}
                        <ul class="space-y-2">
                          {#each regSquads as part, idx}
                            <li class="text-sm text-slate-200 bg-slate-900 px-3 py-2 rounded-xl border border-slate-800 flex items-center justify-between">
                              <span>{idx + 1}. {part.squad?.name || 'Squad'}</span>
                              {#if isSystemAdmin && tourney.status !== 'completed'}
                                <button
                                  type="button"
                                  onclick={() => handleDeclareChampion(tourney.id, part.squad_id)}
                                  class="text-[11px] bg-amber-500/15 hover:bg-amber-500 text-amber-300 hover:text-slate-950 px-2 py-1 rounded-lg font-bold transition cursor-pointer"
                                >
                                  🏆 Set Winner
                                </button>
                              {/if}
                            </li>
                          {/each}
                        </ul>
                      {/if}

                      {#if tourney.status === 'open' && myEligibleSquads.length > 0}
                        <div class="pt-3 border-t border-slate-800 space-y-2.5">
                          <label for="regSquad-{tourney.id}" class="block text-xs text-slate-400 font-semibold">
                            Register Your {tourney.game_type} Squad:
                          </label>
                          <select
                            id="regSquad-{tourney.id}"
                            bind:value={selectedSquadForTourney[tourney.id]}
                            class="w-full bg-slate-900 border border-slate-800 rounded-xl px-3 py-2 text-xs text-white outline-none"
                          >
                            <option value="">-- Select Your Squad --</option>
                            {#each myEligibleSquads as sq}
                              <option value={sq.id}>{sq.name}</option>
                            {/each}
                          </select>
                          <button
                            type="button"
                            onclick={() => handleRegisterTournament(tourney.id, tourney.game_type)}
                            class="w-full bg-purple-600 hover:bg-purple-500 text-white font-bold py-2.5 rounded-xl text-xs transition cursor-pointer"
                          >
                            Register Squad Now
                          </button>
                        </div>
                      {/if}

                      {#if isSystemAdmin && tourney.status === 'open'}
                        <div class="pt-3 border-t border-slate-800">
                          <button
                            type="button"
                            onclick={() => handleGenerateRandomBracket(tourney.id)}
                            class="w-full bg-emerald-500 hover:bg-emerald-400 text-slate-950 font-extrabold py-2.5 rounded-xl text-xs flex items-center justify-center space-x-2 transition cursor-pointer"
                          >
                            <Shuffle class="w-4 h-4" />
                            <span>Generate Random Bracket</span>
                          </button>
                        </div>
                      {/if}
                    </div>

                    <div class="lg:col-span-2 bg-slate-950/70 border border-slate-800/80 rounded-2xl p-5 space-y-4">
                      <h4 class="text-sm font-bold text-white uppercase tracking-wider flex items-center space-x-2">
                        <Swords class="w-4 h-4 text-purple-400" />
                        <span>Tournament Matchups (Random Bracket)</span>
                      </h4>

                      {#if tMatches.length === 0}
                        <div class="text-center py-12 text-slate-400 text-xs">
                          အသင်းများ စာရင်းသွင်းပြီးချိန်တွင် Admin မှ Random ပွဲစဉ်ဇယား ထုတ်ပေးပါမည်။
                        </div>
                      {:else}
                        <div class="grid grid-cols-1 md:grid-cols-2 gap-4">
                          {#each tMatches as match, mIdx}
                            <div class="bg-slate-900 border border-slate-800 p-4 rounded-2xl space-y-3">
                              <div class="flex justify-between items-center text-xs text-slate-400">
                                <span class="font-bold text-purple-400">Match #{mIdx + 1} (Round {match.round_number})</span>
                                <span class="uppercase">{match.status}</span>
                              </div>

                              <div class="space-y-2">
                                <div class="flex justify-between items-center bg-slate-950 px-3 py-2 rounded-xl border {match.winner_id === match.squad1_id ? 'border-emerald-500/60 text-emerald-400 font-bold' : 'border-slate-800 text-slate-200'} text-sm">
                                  <span>{match.squad1?.name || 'TBD'}</span>
                                  {#if isSystemAdmin && match.status !== 'completed' && match.squad1_id}
                                    <button
                                      type="button"
                                      onclick={() => handleSetMatchWinner(match.id, match.squad1_id)}
                                      class="text-[10px] bg-slate-800 hover:bg-emerald-500 hover:text-slate-950 px-2 py-0.5 rounded font-bold cursor-pointer"
                                    >
                                      Win
                                    </button>
                                  {:else if match.winner_id === match.squad1_id}
                                    <span class="text-xs">✓</span>
                                  {/if}
                                </div>

                                <div class="flex justify-between items-center bg-slate-950 px-3 py-2 rounded-xl border {match.winner_id === match.squad2_id ? 'border-emerald-500/60 text-emerald-400 font-bold' : 'border-slate-800 text-slate-200'} text-sm">
                                  <span>{match.squad2?.name || 'BYE (Auto Advance)'}</span>
                                  {#if isSystemAdmin && match.status !== 'completed' && match.squad2_id}
                                    <button
                                      type="button"
                                      onclick={() => handleSetMatchWinner(match.id, match.squad2_id)}
                                      class="text-[10px] bg-slate-800 hover:bg-emerald-500 hover:text-slate-950 px-2 py-0.5 rounded font-bold cursor-pointer"
                                    >
                                      Win
                                    </button>
                                  {:else if match.winner_id === match.squad2_id}
                                    <span class="text-xs">✓</span>
                                  {/if}
                                </div>
                              </div>
                            </div>
                          {/each}
                        </div>
                      {/if}
                    </div>
                  </div>
                </div>
              {/each}
            </div>
          {/if}
        </div>

      <!-- ================================= -->
      <!-- 5. ACTIVE MEMBERS DIRECTORY -->
      <!-- ================================= -->
      {:else if activeTab === 'members'}
        <div class="space-y-8">
          <div class="bg-[#0b0f1a] border border-slate-800/90 p-6 rounded-3xl space-y-4">
            <div class="flex flex-col md:flex-row justify-between items-start md:items-center gap-4">
              <div>
                <h2 class="text-2xl font-extrabold text-white">Active Campus Players Directory</h2>
                <p class="text-slate-400 text-sm mt-1">
                  နာမည်တူနေလျှင်တောင် Unique Student ID (TNT-XXXX) ဖြင့် ခွဲခြားနိုင်ပြီး တိုက်ရိုက် Direct Message ပို့နိုင်ပါသည်။
                </p>
              </div>
              <button
                type="button"
                onclick={openProfileModal}
                class="bg-purple-600 hover:bg-purple-500 text-white px-4 py-2.5 rounded-xl text-xs font-bold cursor-pointer"
              >
                Update My Game IDs
              </button>
            </div>

            <div class="grid grid-cols-1 md:grid-cols-3 gap-3">
              <div class="md:col-span-2 relative">
                <Search class="w-4 h-4 text-slate-400 absolute left-3.5 top-3" />
                <input
                  type="text"
                  bind:value={memberSearchQuery}
                  placeholder="Search player name, TNT-XXXX, MLBB role, PUBG ID..."
                  class="w-full bg-slate-950 border border-slate-800 rounded-xl pl-10 pr-4 py-2.5 text-sm text-white outline-none focus:border-purple-500"
                />
              </div>
              <select
                bind:value={memberGameFilter}
                class="bg-slate-950 border border-slate-800 text-slate-200 text-sm font-semibold rounded-xl px-4 py-2.5 outline-none cursor-pointer"
              >
                <option value="ALL">All Players ({allProfiles.length})</option>
                <option value="MLBB">Has MLBB ID</option>
                <option value="PUBG">Has PUBG ID</option>
                <option value="FOOTBALL">Has Futsal Position</option>
              </select>
            </div>
          </div>

          {#if loadingMembers}
            <div class="text-center py-20 text-slate-400">Loading campus players...</div>
          {:else if filteredMembers.length === 0}
            <div class="bg-[#0b0f1a] border border-slate-800 rounded-3xl p-16 text-center text-slate-400">
              No matching players found.
            </div>
          {:else}
            <div class="grid grid-cols-1 md:grid-cols-2 lg:grid-cols-3 gap-5">
              {#each filteredMembers as player (player.id)}
                <div class="bg-[#0b0f1a] border border-slate-800/90 hover:border-purple-500/40 rounded-3xl p-5 flex flex-col justify-between space-y-4">
                  <div class="space-y-4">
                    <div class="flex items-start justify-between gap-2">
                      <div class="flex items-center space-x-3">
                        <div class="relative">
                          <div class="w-12 h-12 rounded-2xl bg-linear-to-br from-purple-600 to-indigo-600 flex items-center justify-center text-white font-black text-lg">
                            {(player.full_name || 'U')[0].toUpperCase()}
                          </div>
                          <span class="w-3 h-3 rounded-full bg-emerald-400 border-2 border-[#0b0f1a] absolute -bottom-0.5 -right-0.5"></span>
                        </div>
                        <div>
                          <h3 class="text-base font-extrabold text-white">{player.full_name || 'UIT Student'}</h3>
                          <span class="text-[11px] font-mono text-purple-300 bg-purple-500/10 px-2 py-0.5 rounded border border-purple-500/20 inline-block mt-0.5">
                            {getUniquePlayerTag(player)}
                          </span>
                        </div>
                      </div>

                      <!-- Multi-Role Badges for Directory Card -->
                      <div class="flex flex-wrap justify-end gap-1">
                        {#each getPlayerRoleBadges(player) as b}
                          <span class="px-2 py-0.5 rounded-full text-[9px] font-extrabold uppercase border {b.classes}">
                            {b.label}
                          </span>
                        {/each}
                      </div>
                    </div>

                    <div class="bg-slate-950/80 border border-slate-800/80 rounded-2xl p-3.5 space-y-1.5 text-xs">
                      <div class="flex justify-between">
                        <span class="text-slate-500 font-semibold">MLBB:</span>
                        <span class="text-blue-400 font-bold">{player.mlbb_id || 'Not set'}</span>
                      </div>
                      <div class="flex justify-between">
                        <span class="text-slate-500 font-semibold">PUBG:</span>
                        <span class="text-amber-400 font-bold">{player.pubg_id || 'Not set'}</span>
                      </div>
                      <div class="flex justify-between">
                        <span class="text-slate-500 font-semibold">Futsal:</span>
                        <span class="text-emerald-400 font-bold">{player.football_position || 'Not set'}</span>
                      </div>
                    </div>
                  </div>

                  <!-- Direct Message Action Button -->
                  <div class="pt-2">
                    {#if currentUser && player.id === currentUser.id}
                      <button
                        type="button"
                        onclick={openProfileModal}
                        class="w-full bg-slate-900 hover:bg-slate-800 text-slate-300 border border-slate-800 py-2.5 rounded-xl text-xs font-bold cursor-pointer"
                      >
                        You (Edit Profile)
                      </button>
                    {:else}
                      <button
                        type="button"
                        onclick={() => startDirectChatWith(player.id)}
                        class="w-full bg-linear-to-r from-purple-600/20 to-violet-600/20 hover:from-purple-600 hover:to-violet-600 text-purple-300 hover:text-white border border-purple-500/30 py-2.5 rounded-xl text-xs font-extrabold transition flex items-center justify-center space-x-2 cursor-pointer"
                      >
                        <Mail class="w-3.5 h-3.5" />
                        <span>Direct Message (စာပို့မည်)</span>
                      </button>
                    {/if}
                  </div>
                </div>
              {/each}
            </div>
          {/if}
        </div>

      <!-- ================================= -->
      <!-- 5.5 DIRECT MESSAGES (1-ON-1 PLAYER CHAT) -->
      <!-- ================================= -->
      {:else if activeTab === 'messages'}
        {@const activePartner = allProfiles.find((p) => p.id === selectedDmPartnerId)}

        {#if !currentUser}
          <div class="bg-[#0b0f1a] border border-slate-800 rounded-3xl p-16 text-center space-y-4">
            <Mail class="w-12 h-12 text-purple-400 mx-auto" />
            <h3 class="text-lg font-bold text-white">Sign In Required for Direct Messages</h3>
            <button
              type="button"
              onclick={() => (showAuthModal = true)}
              class="bg-purple-600 text-white font-bold px-6 py-2.5 rounded-xl text-sm cursor-pointer"
            >
              Sign In Now
            </button>
          </div>
        {:else}
          <div class="grid grid-cols-1 lg:grid-cols-3 gap-6 h-155">

            <!-- LEFT SIDEBAR: PLAYER CONVERSATION LIST -->
            <div class="bg-[#0b0f1a] border border-slate-800/90 rounded-3xl p-4 flex flex-col h-full">
              <div class="pb-3 border-b border-slate-800 space-y-2.5">
                <h3 class="text-sm font-extrabold uppercase tracking-wider text-purple-300 px-1">
                  Direct Messages (Players)
                </h3>
                <div class="relative">
                  <Search class="w-3.5 h-3.5 text-slate-400 absolute left-3 top-3" />
                  <input
                    type="text"
                    bind:value={dmSearchQuery}
                    placeholder="Search player name or TNT-XXXX..."
                    class="w-full bg-slate-950 border border-slate-800 rounded-xl pl-8 pr-3 py-2 text-xs text-white outline-none focus:border-purple-500"
                  />
                </div>
              </div>

              <div class="flex-1 overflow-y-auto py-3 space-y-1.5 pr-1">
                {#if dmPartnerList.length === 0}
                  <p class="text-xs text-slate-500 text-center py-10">No other players found.</p>
                {:else}
                  {#each dmPartnerList as p (p.id)}
                    {@const lastMsg = directMessages
                      .filter(
                        (m) =>
                          (m.sender_id === currentUser.id && m.receiver_id === p.id) ||
                          (m.sender_id === p.id && m.receiver_id === currentUser.id)
                      )
                      .slice(-1)[0]}

                    <button
                      type="button"
                      onclick={() => (selectedDmPartnerId = p.id)}
                      class="w-full text-left p-3 rounded-2xl transition flex items-center justify-between gap-2 cursor-pointer {selectedDmPartnerId === p.id ? 'bg-purple-600/25 border border-purple-500/50' : 'bg-slate-950/60 hover:bg-slate-900 border border-slate-800/60'}"
                    >
                      <div class="flex items-center space-x-3 min-w-0">
                        <div class="w-9 h-9 rounded-xl bg-linear-to-br from-purple-600 to-indigo-600 text-white font-black flex items-center justify-center text-xs shrink-0">
                          {(p.full_name || 'U')[0].toUpperCase()}
                        </div>
                        <div class="min-w-0">
                          <div class="flex items-center space-x-1.5">
                            <span class="text-xs font-extrabold text-white truncate">
                              {p.full_name || 'UIT Student'}
                            </span>
                            <span class="text-[10px] font-mono text-purple-400 shrink-0">
                              {getUniquePlayerTag(p)}
                            </span>
                          </div>
                          <p class="text-[11px] text-slate-400 truncate mt-0.5">
                            {lastMsg ? lastMsg.content : p.mlbb_id ? `MLBB: ${p.mlbb_id}` : 'Click to start chat'}
                          </p>
                        </div>
                      </div>
                    </button>
                  {/each}
                {/if}
              </div>
            </div>

            <!-- RIGHT AREA: 1-ON-1 CHAT WINDOW -->
            <div class="lg:col-span-2 bg-[#0b0f1a] border border-slate-800/90 rounded-3xl p-6 flex flex-col justify-between h-full">
              {#if !activePartner}
                <div class="flex-1 flex flex-col items-center justify-center text-center space-y-3">
                  <Mail class="w-12 h-12 text-slate-700" />
                  <h4 class="text-base font-bold text-slate-300">Select a Player to Chat</h4>
                  <p class="text-xs text-slate-500 max-w-xs">
                    ဘယ်ဘက်စာရင်းမှ ကစားသမားတစ်ဦးကို ရွေးချယ်ပြီး သီးသန့် (1-on-1) စကားပြောနိုင်ပါသည်။
                  </p>
                </div>
              {:else}
                <!-- Chat Partner Header -->
                <div class="flex items-center justify-between border-b border-slate-800 pb-4">
                  <div class="flex items-center space-x-3">
                    <div class="w-10 h-10 rounded-xl bg-linear-to-br from-purple-600 to-violet-600 text-white font-black flex items-center justify-center text-sm">
                      {(activePartner.full_name || 'U')[0].toUpperCase()}
                    </div>
                    <div>
                      <div class="flex items-center space-x-2">
                        <h4 class="text-base font-extrabold text-white">{activePartner.full_name}</h4>
                        <span class="text-[10px] font-mono bg-purple-500/15 text-purple-300 px-2 py-0.5 rounded border border-purple-500/30">
                          {getUniquePlayerTag(activePartner)}
                        </span>
                      </div>
                      <span class="text-[11px] text-slate-400">
                        {#if activePartner.mlbb_id}MLBB: {activePartner.mlbb_id} {/if}
                        {#if activePartner.pubg_id}• PUBG: {activePartner.pubg_id}{/if}
                      </span>
                    </div>
                  </div>
                </div>

                <!-- Messages Stream -->
                <div class="flex-1 overflow-y-auto py-4 space-y-3 pr-1">
                  {#if currentDmConversation.length === 0}
                    <div class="text-center py-20 space-y-2">
                      <p class="text-sm font-bold text-slate-400">
                        No messages with {activePartner.full_name} yet
                      </p>
                      <p class="text-xs text-slate-500">
                        Send a direct message below to invite them or coordinate a match!
                      </p>
                    </div>
                  {:else}
                    {#each currentDmConversation as dm (dm.id)}
                      {@const isMe = dm.sender_id === currentUser.id}
                      <div class="flex flex-col {isMe ? 'items-end' : 'items-start'}">
                        <span class="text-[10px] text-slate-500 mb-1 px-1">
                          {isMe ? 'You' : activePartner.full_name} • {new Date(dm.created_at).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })}
                        </span>
                        <div class="max-w-[75%] px-4 py-2.5 rounded-2xl text-sm {isMe ? 'bg-linear-to-r from-purple-600 to-violet-600 text-white font-semibold rounded-br-xs' : 'bg-slate-950 border border-slate-800 text-slate-200 rounded-bl-xs'}">
                          {dm.content}
                        </div>
                      </div>
                    {/each}
                  {/if}
                </div>

                <!-- Input Form -->
                <form onsubmit={handleSendDirectMessage} class="pt-3 border-t border-slate-800 space-y-2">
                  <div class="flex items-center space-x-1.5">
                    {#each quickEmojis as emoji}
                      <button
                        type="button"
                        onclick={() => (newDmText = `${newDmText} ${emoji}`.trim())}
                        class="bg-slate-950 hover:bg-slate-800 border border-slate-800 px-2.5 py-1 rounded-lg text-xs cursor-pointer"
                      >
                        {emoji}
                      </button>
                    {/each}
                  </div>
                  <div class="flex items-center space-x-2.5">
                    <input
                      type="text"
                      bind:value={newDmText}
                      placeholder="Message {activePartner.full_name} ({getUniquePlayerTag(activePartner)})..."
                      maxlength="400"
                      class="flex-1 bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-sm text-white outline-none focus:border-purple-500"
                    />
                    <button
                      type="submit"
                      class="bg-linear-to-r from-purple-600 to-violet-600 hover:from-purple-500 hover:to-violet-500 text-white font-extrabold px-5 py-3 rounded-xl transition cursor-pointer flex items-center space-x-1.5"
                    >
                      <Send class="w-4 h-4" />
                      <span>Send</span>
                    </button>
                  </div>
                </form>
              {/if}
            </div>

          </div>
        {/if}

      <!-- ================================= -->
      <!-- 6. ALL CHAT (GLOBAL COMMUNITY CHAT) -->
      <!-- ================================= -->
      {:else if activeTab === 'allchat'}
        <div class="bg-[#0b0f1a] border border-slate-800/90 rounded-3xl p-6 flex flex-col justify-between h-155">
          <div class="flex items-center justify-between border-b border-slate-800 pb-4">
            <div class="flex items-center space-x-3">
              <div class="w-10 h-10 rounded-2xl bg-purple-600/20 border border-purple-500/40 flex items-center justify-center text-purple-400">
                <MessageSquare class="w-5 h-5" />
              </div>
              <div>
                <h3 class="text-lg font-extrabold text-white flex items-center space-x-2">
                  <span>UIT Global All Chat</span>
                  <span class="px-2 py-0.5 text-[10px] rounded-full bg-emerald-500/15 text-emerald-400 font-bold">
                    LIVE
                  </span>
                </h3>
                <p class="text-xs text-slate-400">Find teammates, call for 5v5 rooms, or chat with campus players.</p>
              </div>
            </div>
          </div>

          <div class="flex-1 overflow-y-auto py-4 space-y-3 pr-1">
            {#if globalMessages.length === 0}
              <div class="text-center py-24 space-y-2">
                <MessageSquare class="w-10 h-10 text-slate-700 mx-auto" />
                <p class="text-sm font-bold text-slate-400">Welcome to UIT Global All Chat!</p>
                <p class="text-xs text-slate-500">Start the conversation or post a recruitment call below.</p>
              </div>
            {:else}
              {#each globalMessages as msg (msg.id)}
                {@const isMe = currentUser && msg.user_id === currentUser.id}
                <div class="flex flex-col {isMe ? 'items-end' : 'items-start'} group">
                  <div class="flex items-center space-x-2 mb-1 px-1">
                    <span class="text-[11px] font-bold text-slate-300">
                      {msg.profiles?.full_name || 'UIT Player'}
                    </span>
                    {#if msg.profiles?.student_id}
                      <span class="text-[10px] font-mono text-purple-400">({msg.profiles.student_id})</span>
                    {/if}
                    <span class="text-[10px] text-slate-500">
                      {new Date(msg.created_at).toLocaleTimeString([], { hour: '2-digit', minute: '2-digit' })}
                    </span>
                    <button
                      type="button"
                      onclick={() => (replyToMessage = msg)}
                      class="opacity-0 group-hover:opacity-100 text-[10px] text-purple-400 hover:underline flex items-center space-x-0.5 cursor-pointer"
                    >
                      <Reply class="w-3 h-3" />
                      <span>Reply</span>
                    </button>
                  </div>
                  <div class="max-w-[75%] px-4 py-2.5 rounded-2xl text-sm {isMe ? 'bg-linear-to-r from-purple-600 to-violet-600 text-white font-semibold rounded-br-xs' : 'bg-slate-950 border border-slate-800 text-slate-200 rounded-bl-xs'}">
                    {msg.content}
                  </div>
                </div>
              {/each}
            {/if}
          </div>

          <form onsubmit={handleSendGlobalMessage} class="pt-3 border-t border-slate-800 space-y-2">
            {#if replyToMessage}
              <div class="bg-slate-950 border border-purple-500/30 px-3 py-1.5 rounded-xl flex items-center justify-between text-xs">
                <span class="text-slate-400 truncate">
                  Replying to <strong class="text-purple-300">{replyToMessage.profiles?.full_name || 'Player'}</strong>: {replyToMessage.content}
                </span>
                <button
                  type="button"
                  onclick={() => (replyToMessage = null)}
                  class="text-slate-400 hover:text-white ml-2 cursor-pointer"
                >
                  <X class="w-3.5 h-3.5" />
                </button>
              </div>
            {/if}

            <div class="flex items-center space-x-1.5">
              {#each quickEmojis as emoji}
                <button
                  type="button"
                  onclick={() => (newGlobalMessage = `${newGlobalMessage} ${emoji}`.trim())}
                  class="bg-slate-950 hover:bg-slate-800 border border-slate-800 px-2.5 py-1 rounded-lg text-xs cursor-pointer"
                >
                  {emoji}
                </button>
              {/each}
            </div>

            <div class="flex items-center space-x-2.5">
              <input
                type="text"
                bind:value={newGlobalMessage}
                placeholder="Send a message to everyone in UIT Arena..."
                maxlength="400"
                class="flex-1 bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-sm text-white outline-none focus:border-purple-500"
              />
              <button
                type="submit"
                class="bg-linear-to-r from-purple-600 to-violet-600 hover:from-purple-500 hover:to-violet-500 text-white font-extrabold px-6 py-3 rounded-xl transition cursor-pointer flex items-center space-x-1.5"
              >
                <Send class="w-4 h-4" />
                <span>Send</span>
              </button>
            </div>
          </form>
        </div>

      <!-- ================================= -->
      <!-- 7. SQUAD, PLAYER & ADMIN DASHBOARD -->
      <!-- ================================= -->
      {:else if activeTab === 'dashboard'}
        <div class="space-y-8">
          <div class="bg-linear-to-r from-[#0b0f1a] via-purple-950/30 to-[#0b0f1a] border border-purple-500/30 p-8 rounded-3xl flex flex-col md:flex-row justify-between items-start md:items-center gap-4">
            <div>
              <div class="flex flex-wrap items-center gap-1.5 mb-2">
                <span class="text-xs font-bold uppercase tracking-widest text-slate-400 mr-1">
                  Active Roles:
                </span>
                {#if isSystemAdmin}
                  <span class="px-2.5 py-0.5 rounded-full text-[10px] font-extrabold uppercase bg-amber-500/20 text-amber-300 border border-amber-500/40">
                    👑 System Admin (Exclusive)
                  </span>
                {/if}
                {#if isAnySquadLeader}
                  <span class="px-2.5 py-0.5 rounded-full text-[10px] font-extrabold uppercase bg-purple-500/20 text-purple-300 border border-purple-500/40">
                    🛡️ Squad Leader ({mySquads.length})
                  </span>
                {/if}
                {#if isAnySquadMember || (!isSystemAdmin && !isAnySquadLeader)}
                  <span class="px-2.5 py-0.5 rounded-full text-[10px] font-extrabold uppercase bg-emerald-500/15 text-emerald-300 border border-emerald-500/30">
                    🎮 Squad Member
                  </span>
                {/if}
              </div>
              <h2 class="text-2xl font-extrabold text-white">Unified Command & Player Dashboard</h2>
              <p class="text-slate-400 text-sm mt-1">
                All your active roles work together seamlessly here.
              </p>
            </div>

            <div class="flex flex-wrap gap-3">
              {#if isSystemAdmin}
                <button
                  type="button"
                  onclick={() => (showTourneyModal = true)}
                  class="bg-amber-500 hover:bg-amber-400 text-slate-950 font-extrabold px-4 py-2.5 rounded-xl text-xs cursor-pointer flex items-center space-x-1.5"
                >
                  <Trophy class="w-4 h-4" />
                  <span>+ Launch Tournament</span>
                </button>
              {/if}
              <button
                type="button"
                onclick={openProfileModal}
                class="bg-slate-900 hover:bg-slate-800 text-purple-300 font-bold px-4 py-2.5 rounded-xl text-xs cursor-pointer border border-purple-500/30 flex items-center space-x-1.5"
              >
                <User class="w-4 h-4" />
                <span>Edit Profile</span>
              </button>
              <button
                type="button"
                onclick={openCreateModal}
                class="bg-linear-to-r from-purple-600 to-violet-600 text-white font-bold px-4 py-2.5 rounded-xl text-xs cursor-pointer flex items-center space-x-1.5"
              >
                <Plus class="w-4 h-4" />
                <span>New Squad</span>
              </button>
            </div>
          </div>

          <!-- ========================================== -->
          <!-- ADMIN COMMAND CENTER (VISIBLE ONLY TO YOUR ACCOUNT) -->
          <!-- ========================================== -->
          {#if isSystemAdmin}
            <div class="bg-[#0b0f1a] border-2 border-amber-500/40 rounded-3xl p-6 space-y-6 shadow-2xl shadow-amber-500/5">
              <div class="flex flex-col sm:flex-row justify-between items-start sm:items-center gap-3 border-b border-slate-800 pb-4">
                <div class="flex items-center space-x-3">
                  <div class="w-10 h-10 rounded-2xl bg-amber-500/20 border border-amber-500/40 flex items-center justify-center text-amber-400">
                    <ShieldAlert class="w-5 h-5" />
                  </div>
                  <div>
                    <h3 class="text-lg font-black text-white">Exclusive Admin Command Center 👑</h3>
                    <p class="text-xs text-slate-400">
                      Only your Master Admin account ({MASTER_ADMIN_EMAIL}) has access to this panel.
                    </p>
                  </div>
                </div>
                <button
                  type="button"
                  onclick={() => (showTourneyModal = true)}
                  class="bg-amber-500 hover:bg-amber-400 text-slate-950 font-extrabold px-4 py-2 rounded-xl text-xs cursor-pointer"
                >
                  + Create Official Tournament
                </button>
              </div>

              <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
                <!-- Registered Players Inspection -->
                <div class="bg-slate-950/80 border border-slate-800 rounded-2xl p-4 space-y-3">
                  <h4 class="text-xs font-extrabold uppercase tracking-wider text-amber-400">
                    Registered Campus Players ({allProfiles.length})
                  </h4>
                  <div class="space-y-2 max-h-64 overflow-y-auto pr-1">
                    {#each allProfiles as p (p.id)}
                      <div class="bg-slate-900 border border-slate-800 px-3.5 py-2.5 rounded-xl flex items-center justify-between gap-2">
                        <div class="min-w-0">
                          <span class="text-xs font-bold text-white block truncate">
                            {p.full_name || 'UIT Player'}
                          </span>
                          <span class="text-[10px] font-mono text-purple-400">
                            {getUniquePlayerTag(p)} • UUID: {p.id.slice(0, 8)}...
                          </span>
                        </div>
                        <div class="flex items-center space-x-1 shrink-0">
                          {#each getPlayerRoleBadges(p) as b}
                            <span class="px-2 py-0.5 rounded-md text-[9px] font-extrabold uppercase border {b.classes}">
                              {b.label}
                            </span>
                          {/each}
                        </div>
                      </div>
                    {/each}
                  </div>
                </div>

                <!-- Admin Squad Moderation -->
                <div class="bg-slate-950/80 border border-slate-800 rounded-2xl p-4 space-y-3">
                  <h4 class="text-xs font-extrabold uppercase tracking-wider text-amber-400">
                    All Campus Squads Moderation ({squads.length})
                  </h4>
                  <div class="space-y-2 max-h-64 overflow-y-auto pr-1">
                    {#each squads as sq (sq.id)}
                      <div class="bg-slate-900 border border-slate-800 px-3.5 py-2.5 rounded-xl flex items-center justify-between gap-2">
                        <div>
                          <span class="text-xs font-bold text-white block">
                            {sq.name} <span class="text-purple-400">({sq.game_type})</span>
                          </span>
                          <span class="text-[10px] text-slate-400">
                            Leader: {sq.profiles?.full_name || 'Leader'} • {sq.wins ?? 0}W - {sq.losses ?? 0}L
                          </span>
                        </div>
                        <button
                          type="button"
                          onclick={() => handleDisbandSquad(sq.id, sq.name)}
                          class="bg-red-500/15 hover:bg-red-500 text-red-400 hover:text-white px-2.5 py-1 rounded-lg text-[11px] font-bold transition cursor-pointer"
                        >
                          Delete Squad
                        </button>
                      </div>
                    {/each}
                  </div>
                </div>
              </div>
            </div>
          {/if}

          <div class="grid grid-cols-1 lg:grid-cols-2 gap-6">
            <div class="bg-[#0b0f1a] border border-slate-800/90 rounded-3xl p-6 space-y-4">
              <h3 class="text-lg font-bold text-white flex items-center space-x-2">
                <UserCheck class="w-5 h-5 text-purple-400" />
                <span>Incoming Member Join Requests (For Squads You Lead)</span>
              </h3>

              {#if pendingMembers.length === 0}
                <p class="text-sm text-slate-400 py-6 text-center">
                  လက်ရှိတွင် သင်ဦးဆောင်သော Squad သို့ ဝင်ရောက်ရန် တောင်းဆိုထားသူ မရှိသေးပါ။
                </p>
              {:else}
                <div class="space-y-3">
                  {#each pendingMembers as req (req.id)}
                    <div class="bg-slate-950 border border-slate-800 p-4 rounded-2xl flex items-center justify-between gap-4">
                      <div>
                        <div class="flex items-center space-x-2">
                          <span class="text-sm font-bold text-white">
                            {req.profiles?.full_name || 'UIT Student'}
                          </span>
                          <span class="text-[10px] font-mono text-purple-400">
                            {getUniquePlayerTag(req.profiles)}
                          </span>
                        </div>
                        <span class="text-xs text-slate-400 block">
                          Applying for: <strong class="text-purple-300">{req.squads?.name} ({req.squads?.game_type})</strong>
                        </span>
                      </div>

                      <div class="flex items-center space-x-2">
                        {#if req.status === 'pending'}
                          <button
                            type="button"
                            onclick={() => handleMemberStatus(req, 'approved')}
                            class="bg-emerald-500 hover:bg-emerald-400 text-slate-950 px-3 py-1.5 rounded-xl text-xs font-bold flex items-center space-x-1 cursor-pointer"
                          >
                            <Check class="w-3.5 h-3.5" />
                            <span>Accept</span>
                          </button>
                          <button
                            type="button"
                            onclick={() => handleMemberStatus(req, 'rejected')}
                            class="bg-red-500/20 hover:bg-red-500 text-red-400 hover:text-white px-3 py-1.5 rounded-xl text-xs font-bold flex items-center space-x-1 cursor-pointer"
                          >
                            <XCircle class="w-3.5 h-3.5" />
                            <span>Reject</span>
                          </button>
                        {:else}
                          <span class="text-xs font-bold uppercase px-2.5 py-1 rounded-lg {req.status === 'approved' ? 'bg-emerald-500/10 text-emerald-400' : 'bg-red-500/10 text-red-400'}">
                            {req.status}
                          </span>
                        {/if}
                      </div>
                    </div>
                  {/each}
                </div>
              {/if}
            </div>

            <div class="bg-[#0b0f1a] border border-slate-800/90 rounded-3xl p-6 space-y-4">
              <h3 class="text-lg font-bold text-white flex items-center space-x-2">
                <Users class="w-5 h-5 text-emerald-400" />
                <span>My Squad Applications (As Member)</span>
              </h3>

              {#if myJoinedSquads.length === 0}
                <p class="text-sm text-slate-400 py-6 text-center">
                  သင် Member အဖြစ် လျှောက်ထားသော Squad မရှိသေးပါ။
                </p>
              {:else}
                <div class="space-y-3">
                  {#each myJoinedSquads as item (item.id)}
                    <div class="bg-slate-950 border border-slate-800 p-4 rounded-2xl flex items-center justify-between gap-2">
                      <div>
                        <span class="text-sm font-bold text-white block">{item.squads?.name}</span>
                        <span class="text-xs text-slate-400">{item.squads?.game_type}</span>
                      </div>
                      <div class="flex items-center space-x-2">
                        <span class="text-xs font-bold uppercase px-3 py-1 rounded-lg {item.status === 'approved' ? 'bg-emerald-500/10 text-emerald-400' : item.status === 'rejected' ? 'bg-red-500/10 text-red-400' : 'bg-amber-500/10 text-amber-400'}">
                          {item.status}
                        </span>
                        {#if item.status === 'approved'}
                          <button
                            type="button"
                            onclick={() => {
                              selectedMySquadId = item.squad_id;
                              activeTab = 'mysquad';
                              fetchMySquadDetails(item.squad_id);
                            }}
                            class="bg-purple-600 text-white px-3 py-1 rounded-lg text-xs font-bold cursor-pointer"
                          >
                            Open HQ →
                          </button>
                        {/if}
                      </div>
                    </div>
                  {/each}
                </div>
              {/if}
            </div>
          </div>
        </div>
      {/if}

    </main>
  </div>

  <!-- ================================= -->
  <!-- CUSTOM CONFIRMATION DIALOG MODAL -->
  <!-- ================================= -->
  {#if confirmDialog.open}
    <div class="fixed inset-0 bg-black/80 backdrop-blur-md flex items-center justify-center z-90 p-4">
      <div class="bg-[#0d121f] border border-purple-500/30 w-full max-w-sm rounded-3xl p-6 space-y-4 shadow-2xl">
        <h3 class="text-lg font-extrabold text-white">{confirmDialog.title}</h3>
        <p class="text-xs text-slate-300 leading-relaxed">{confirmDialog.message}</p>
        <div class="flex justify-end space-x-2.5 pt-2">
          <button
            type="button"
            onclick={() => (confirmDialog.open = false)}
            class="bg-slate-800 hover:bg-slate-700 text-slate-300 px-4 py-2 rounded-xl text-xs font-bold cursor-pointer"
          >
            Cancel
          </button>
          <button
            type="button"
            onclick={() => {
              confirmDialog.open = false;
              confirmDialog.onConfirm();
            }}
            class="bg-red-500 hover:bg-red-600 text-white px-4 py-2 rounded-xl text-xs font-extrabold cursor-pointer"
          >
            Confirm
          </button>
        </div>
      </div>
    </div>
  {/if}

  <!-- ================================= -->
  <!-- MODAL 0: PLAYER PROFILE & MULTI-ROLE BADGES (NO PUBLIC ADMIN TOGGLE) -->
  <!-- ================================= -->
  {#if showProfileModal}
    <div class="fixed inset-0 bg-black/80 backdrop-blur-md flex items-center justify-center z-50 p-4 overflow-y-auto">
      <div class="bg-[#0b0f1a] border border-purple-500/30 w-full max-w-lg rounded-3xl p-8 relative shadow-2xl my-8">
        <button
          type="button"
          onclick={() => (showProfileModal = false)}
          class="absolute top-6 right-6 text-slate-400 hover:text-white bg-slate-800/80 p-2 rounded-full cursor-pointer"
        >
          <X class="w-5 h-5" />
        </button>

        <div class="flex items-center space-x-4 mb-6">
          <div class="w-14 h-14 rounded-2xl bg-linear-to-br from-purple-600 to-violet-600 flex items-center justify-center text-white font-black text-2xl shadow-lg">
            {(profileFullName || 'U')[0].toUpperCase()}
          </div>
          <div>
            <h3 class="text-2xl font-bold text-white">{profileFullName || 'UIT Player'}</h3>
            <div class="flex flex-wrap gap-1.5 mt-1">
              {#if isSystemAdmin}
                <span class="px-2.5 py-0.5 rounded-full text-[10px] font-extrabold uppercase bg-amber-500/20 text-amber-300 border border-amber-500/40">
                  👑 System Admin
                </span>
              {/if}
              {#if isAnySquadLeader}
                <span class="px-2.5 py-0.5 rounded-full text-[10px] font-extrabold uppercase bg-purple-500/20 text-purple-300 border border-purple-500/40">
                  🛡️ Squad Leader ({mySquads.length})
                </span>
              {/if}
              {#if isAnySquadMember || (!isSystemAdmin && !isAnySquadLeader)}
                <span class="px-2.5 py-0.5 rounded-full text-[10px] font-extrabold uppercase bg-emerald-500/15 text-emerald-300 border border-emerald-500/30">
                  🎮 Squad Member
                </span>
              {/if}
            </div>
            <p class="text-xs text-slate-400 mt-1.5">{currentUser?.email}</p>
          </div>
        </div>

        <div class="bg-slate-950/80 border border-slate-800 rounded-2xl p-4 mb-5 space-y-2.5">
          <span class="text-[11px] font-extrabold uppercase tracking-wider text-purple-400 block">
            My Active Squads & Context Roles ({activeMySquadsList.length}/3 Games)
          </span>
          {#if activeMySquadsList.length === 0}
            <p class="text-xs text-slate-400">လက်ရှိ ဝင်ရောက်ထားသော Squad မရှိသေးပါ။</p>
          {:else}
            <div class="flex flex-wrap gap-2">
              {#each activeMySquadsList as sq (sq.id)}
                <button
                  type="button"
                  onclick={() => {
                    showProfileModal = false;
                    selectedMySquadId = sq.id;
                    activeTab = 'mysquad';
                    fetchMySquadDetails(sq.id);
                  }}
                  class="bg-slate-900 hover:bg-purple-600 text-slate-200 hover:text-white border border-slate-700 px-3 py-1.5 rounded-xl text-xs font-bold transition flex items-center space-x-1.5 cursor-pointer"
                >
                  <span>
                    {sq.leader_id === currentUser?.id ? '👑 Leader:' : '🎮 Member:'} {sq.name}
                  </span>
                  <span class="text-[10px] opacity-75">({sq.game_type})</span>
                </button>
              {/each}
            </div>
          {/if}
        </div>

        {#if errorMessage}
          <div class="bg-red-500/10 border border-red-500/50 text-red-400 text-xs p-3.5 rounded-xl mb-4 leading-relaxed">
            ⚠️ {errorMessage}
          </div>
        {/if}

        {#if successMessage}
          <div class="bg-emerald-500/10 border border-emerald-500/50 text-emerald-400 text-xs p-3.5 rounded-xl mb-4">
            ✅ {successMessage}
          </div>
        {/if}

        <form onsubmit={handleSaveProfile} class="space-y-4">
          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
            <div>
              <label for="profName" class="block text-[11px] font-bold uppercase text-slate-400 mb-1">
                Full Name (အမည်)
              </label>
              <input
                id="profName"
                type="text"
                bind:value={profileFullName}
                required
                class="w-full bg-slate-950 border border-slate-800 rounded-xl px-3.5 py-2.5 text-white text-sm outline-none focus:border-purple-500"
              />
            </div>

            <div>
              <label for="profSid" class="block text-[11px] font-bold uppercase text-purple-300 mb-1">
                Student ID (Unique • တူ၍မရပါ)
              </label>
              <input
                id="profSid"
                type="text"
                bind:value={profileStudentId}
                required
                placeholder="TNT-XXXX"
                class="w-full bg-slate-950 border border-slate-800 rounded-xl px-3.5 py-2.5 text-white text-sm font-mono uppercase outline-none focus:border-purple-500"
              />
            </div>
          </div>

          <div class="grid grid-cols-1 sm:grid-cols-2 gap-4">
            <div>
              <label for="profMlbb" class="block text-[11px] font-bold uppercase text-blue-400 mb-1">
                MLBB ID & Role
              </label>
              <input
                id="profMlbb"
                type="text"
                bind:value={profileMlbbId}
                placeholder="ဥပမာ - 12345678 (Mage/Roam)"
                class="w-full bg-slate-950 border border-slate-800 rounded-xl px-3.5 py-2.5 text-white text-sm outline-none focus:border-purple-500"
              />
            </div>

            <div>
              <label for="profPubg" class="block text-[11px] font-bold uppercase text-amber-400 mb-1">
                PUBG Mobile ID
              </label>
              <input
                id="profPubg"
                type="text"
                bind:value={profilePubgId}
                placeholder="ဥပမာ - 5123456789"
                class="w-full bg-slate-950 border border-slate-800 rounded-xl px-3.5 py-2.5 text-white text-sm outline-none focus:border-purple-500"
              />
            </div>
          </div>

          <div>
            <label for="profFoot" class="block text-[11px] font-bold uppercase text-emerald-400 mb-1">
              Football / Futsal Position
            </label>
            <input
              id="profFoot"
              type="text"
              bind:value={profileFootballPos}
              placeholder="ဥပမာ - Striker / Midfielder / Goalkeeper"
              class="w-full bg-slate-950 border border-slate-800 rounded-xl px-3.5 py-2.5 text-white text-sm outline-none focus:border-purple-500"
            />
          </div>

          <button
            type="submit"
            disabled={loadingAction}
            class="w-full bg-linear-to-r from-purple-600 to-violet-600 hover:from-purple-500 hover:to-violet-500 text-white font-extrabold py-3 rounded-xl transition cursor-pointer"
          >
            {loadingAction ? 'Saving...' : 'Save Player Profile'}
          </button>
        </form>
      </div>
    </div>
  {/if}

  <!-- ================================= -->
  <!-- MODAL 1: CREATE SQUAD -->
  <!-- ================================= -->
  {#if showCreateModal}
    {@const existingSquadForSelectedGame = getActiveSquadForGame(gameType)}
    <div class="fixed inset-0 bg-black/80 backdrop-blur-md flex items-center justify-center z-50 p-4">
      <div class="bg-[#0b0f1a] border border-purple-500/30 w-full max-w-md rounded-3xl p-8 relative shadow-2xl">
        <button
          type="button"
          onclick={closeCreateModal}
          class="absolute top-6 right-6 text-slate-400 hover:text-white bg-slate-800/80 p-2 rounded-full cursor-pointer"
        >
          <X class="w-5 h-5" />
        </button>

        <h3 class="text-2xl font-bold text-white mb-2">Create Your Squad 🛡️</h3>
        <p class="text-slate-400 text-sm mb-6">
          ဂိမ်းအမျိုးအစား တစ်ခုစီအတွက် Squad (၁) ခုသာ တည်ထောင်/ဝင်ရောက်ခွင့် ရှိပါသည်။
        </p>

        {#if errorMessage}
          <div class="bg-red-500/10 border border-red-500/50 text-red-400 text-sm p-3.5 rounded-xl mb-4">
            {errorMessage}
          </div>
        {/if}

        {#if existingSquadForSelectedGame}
          <div class="bg-amber-500/10 border border-amber-500/40 text-amber-300 text-xs p-3.5 rounded-xl mb-4 leading-relaxed">
            ⚠️ သင်သည် <strong>{gameType}</strong> အတွက် <strong>"{existingSquadForSelectedGame.name}"</strong> အသင်းတွင် ပါဝင်ထားပြီးသား ဖြစ်ပါသည်။ အခြား ဂိမ်းအမျိုးအစားကို ရွေးချယ်ပါ။
          </div>
        {/if}

        <form onsubmit={handleCreateSquad} class="space-y-5">
          <div>
            <label for="gameType" class="block text-xs font-bold uppercase tracking-wider text-slate-400 mb-2">
              Game Category (ကစားမည့်ဂိမ်း)
            </label>
            <select
              id="gameType"
              bind:value={gameType}
              class="w-full bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-white text-sm focus:border-purple-500 outline-none cursor-pointer"
            >
              <option value="MLBB">
                MLBB (Mobile Legends) {getActiveSquadForGame('MLBB') ? '— [Already in Squad]' : ''}
              </option>
              <option value="PUBG">
                PUBG Mobile {getActiveSquadForGame('PUBG') ? '— [Already in Squad]' : ''}
              </option>
              <option value="FOOTBALL">
                Football / Futsal {getActiveSquadForGame('FOOTBALL') ? '— [Already in Squad]' : ''}
              </option>
            </select>
          </div>

          <div>
            <label for="squadName" class="block text-xs font-bold uppercase tracking-wider text-slate-400 mb-2">
              Squad Name (အသင်းနာမည်)
            </label>
            <input
              id="squadName"
              type="text"
              bind:value={squadName}
              required
              maxlength="50"
              placeholder="ဥပမာ - UIT Legends"
              class="w-full bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-white text-sm focus:border-purple-500 outline-none"
            />
          </div>

          <div>
            <label for="requirements" class="block text-xs font-bold uppercase tracking-wider text-slate-400 mb-2">
              Requirements (အသင်းဝင်ရန် လိုအပ်ချက်များ)
            </label>
            <textarea
              id="requirements"
              bind:value={requirements}
              required
              maxlength="500"
              rows="3"
              placeholder="ဥပမာ - Mythic 25 stars+, Roamer အဓိကလိုသည်"
              class="w-full bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-white text-sm focus:border-purple-500 outline-none"
            ></textarea>
          </div>

          <button
            type="submit"
            disabled={loadingAction || !!existingSquadForSelectedGame}
            class="w-full bg-linear-to-r from-purple-600 to-violet-600 hover:from-purple-500 hover:to-violet-500 text-white font-extrabold py-3.5 rounded-xl transition-all shadow-lg shadow-purple-600/25 disabled:opacity-40 disabled:cursor-not-allowed cursor-pointer"
          >
            {loadingAction
              ? 'Creating Squad...'
              : existingSquadForSelectedGame
                ? `Already in ${gameType} Squad`
                : 'Create Squad Now'}
          </button>
        </form>
      </div>
    </div>
  {/if}

  <!-- ================================= -->
  <!-- MODAL 2: CHALLENGE SCRIM -->
  <!-- ================================= -->
  {#if showScrimModal}
    <div class="fixed inset-0 bg-black/80 backdrop-blur-md flex items-center justify-center z-50 p-4">
      <div class="bg-[#0b0f1a] border border-purple-500/30 w-full max-w-md rounded-3xl p-8 relative shadow-2xl">
        <button
          type="button"
          onclick={closeScrimModal}
          class="absolute top-6 right-6 text-slate-400 hover:text-white bg-slate-800/80 p-2 rounded-full cursor-pointer"
        >
          <X class="w-5 h-5" />
        </button>

        <h3 class="text-2xl font-bold text-white mb-2">Challenge Scrim ⚔️</h3>
        <p class="text-slate-400 text-sm mb-6">
          သင်ဦးဆောင်သော အသင်းဖြင့် အခြားအသင်းများကို ချိန်းပွဲ စိန်ခေါ်ပါ။
        </p>

        <form onsubmit={handleCreateScrim} class="space-y-5">
          <div>
            <label for="challengerSquad" class="block text-xs font-bold uppercase tracking-wider text-slate-400 mb-2">
              Your Squad (Challenger)
            </label>
            <select
              id="challengerSquad"
              bind:value={challengerSquadId}
              required
              class="w-full bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-white text-sm outline-none focus:border-purple-500 cursor-pointer"
            >
              <option value="">-- Select Your Squad --</option>
              {#each mySquads as squad (squad.id)}
                <option value={squad.id}>{squad.name} ({squad.game_type})</option>
              {/each}
            </select>
          </div>

          <div>
            <label for="opponentSquad" class="block text-xs font-bold uppercase tracking-wider text-slate-400 mb-2">
              Opponent Squad
            </label>
            <select
              id="opponentSquad"
              bind:value={opponentSquadId}
              required
              class="w-full bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-white text-sm outline-none focus:border-purple-500 cursor-pointer"
            >
              <option value="">-- Select Opponent --</option>
              {#each squads as squad (squad.id)}
                {#if squad.id !== challengerSquadId}
                  <option value={squad.id}>{squad.name} ({squad.game_type})</option>
                {/if}
              {/each}
            </select>
          </div>

          <div>
            <label for="scheduledTime" class="block text-xs font-bold uppercase tracking-wider text-slate-400 mb-2">
              Scheduled Time (ချိန်းဆိုမည့်အချိန်)
            </label>
            <input
              id="scheduledTime"
              type="datetime-local"
              bind:value={scheduledTime}
              required
              class="w-full bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-white text-sm outline-none focus:border-purple-500"
            />
          </div>

          <button
            type="submit"
            class="w-full bg-linear-to-r from-purple-600 to-violet-600 hover:from-purple-500 hover:to-violet-500 text-white font-extrabold py-3.5 rounded-xl transition-all shadow-lg shadow-purple-600/25 cursor-pointer"
          >
            Send Scrim Challenge
          </button>
        </form>
      </div>
    </div>
  {/if}

  <!-- ================================= -->
  <!-- MODAL 3: CREATE TOURNAMENT (ADMIN) -->
  <!-- ================================= -->
  {#if showTourneyModal}
    <div class="fixed inset-0 bg-black/80 backdrop-blur-md flex items-center justify-center z-50 p-4">
      <div class="bg-[#0b0f1a] border border-purple-500/30 w-full max-w-md rounded-3xl p-8 relative shadow-2xl">
        <button
          type="button"
          onclick={() => (showTourneyModal = false)}
          class="absolute top-6 right-6 text-slate-400 hover:text-white bg-slate-800/80 p-2 rounded-full cursor-pointer"
        >
          <X class="w-5 h-5" />
        </button>

        <h3 class="text-2xl font-bold text-white mb-2">Create Official Tournament 🏆</h3>
        <p class="text-slate-400 text-sm mb-6">
          Admin အနေဖြင့် ကျောင်းတွင်းပြိုင်ပွဲအသစ်နှင့် စာရင်းပိတ်ချိန် သတ်မှတ်ပါ။
        </p>

        <form onsubmit={handleCreateTournament} class="space-y-5">
          <div>
            <label for="tourneyTitle" class="block text-xs font-bold uppercase tracking-wider text-slate-400 mb-2">
              Tournament Title (ပြိုင်ပွဲအမည်)
            </label>
            <input
              id="tourneyTitle"
              type="text"
              bind:value={tourneyTitle}
              required
              placeholder="ဥပမာ - UIT Freshers MLBB Cup 2026"
              class="w-full bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-white text-sm outline-none focus:border-purple-500"
            />
          </div>

          <div>
            <label for="tourneyGame" class="block text-xs font-bold uppercase tracking-wider text-slate-400 mb-2">
              Game Category
            </label>
            <select
              id="tourneyGame"
              bind:value={tourneyGameType}
              class="w-full bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-white text-sm outline-none cursor-pointer"
            >
              <option value="MLBB">MLBB (Mobile Legends)</option>
              <option value="PUBG">PUBG Mobile</option>
              <option value="FOOTBALL">Football / Futsal</option>
            </select>
          </div>

          <div>
            <label for="tourneyDeadline" class="block text-xs font-bold uppercase tracking-wider text-slate-400 mb-2">
              Registration Deadline (စာရင်းပိတ်မည့်ရက်)
            </label>
            <input
              id="tourneyDeadline"
              type="datetime-local"
              bind:value={tourneyDeadline}
              required
              class="w-full bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-white text-sm outline-none focus:border-purple-500"
            />
          </div>

          <button
            type="submit"
            disabled={loadingAction}
            class="w-full bg-linear-to-r from-purple-600 to-violet-600 text-white font-extrabold py-3.5 rounded-xl cursor-pointer"
          >
            {loadingAction ? 'Creating...' : 'Launch Tournament'}
          </button>
        </form>
      </div>
    </div>
  {/if}

  <!-- ================================= -->
  <!-- MODAL 4: SIGN IN / SIGN UP (WITH UNIQUE TNT-XXXX STUDENT ID) -->
  <!-- ================================= -->
  {#if showAuthModal}
    <div class="fixed inset-0 bg-black/80 backdrop-blur-md flex items-center justify-center z-50 p-4">
      <div class="bg-[#0b0f1a] border border-purple-500/30 w-full max-w-md rounded-3xl p-8 relative shadow-2xl">
        <button
          type="button"
          onclick={() => (showAuthModal = false)}
          class="absolute top-6 right-6 text-slate-400 hover:text-white bg-slate-800/80 p-2 rounded-full cursor-pointer"
        >
          <X class="w-5 h-5" />
        </button>

        <h3 class="text-2xl font-bold text-white mb-2">
          {isLoginMode ? 'Sign In to UIT Arena' : 'Create Student Account'} 🎮
        </h3>
        <p class="text-slate-400 text-sm mb-6">
          Squad တည်ထောင်ရန်နှင့် ပြိုင်ပွဲများတွင် စာရင်းသွင်းရန် အကောင့်ဝင်ပါ။
        </p>

        {#if errorMessage}
          <div class="bg-red-500/10 border border-red-500/50 text-red-400 text-sm p-3.5 rounded-xl mb-4 leading-relaxed">
            ⚠️ {errorMessage}
          </div>
        {/if}

        {#if successMessage}
          <div class="bg-emerald-500/10 border border-emerald-500/50 text-emerald-400 text-sm p-3.5 rounded-xl mb-4 leading-relaxed">
            ✅ {successMessage}
          </div>
        {/if}

        <form onsubmit={handleAuth} class="space-y-4">
          {#if !isLoginMode}
            <div>
              <label for="fullName" class="block text-xs font-bold uppercase text-slate-400 mb-1.5">
                Full Name (အမည်)
              </label>
              <input
                id="fullName"
                type="text"
                bind:value={authFullName}
                required
                placeholder="ဥပမာ - Kyal Sin"
                class="w-full bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-white text-sm outline-none focus:border-purple-500"
              />
            </div>

            <div>
              <label for="studentId" class="block text-xs font-bold uppercase text-purple-300 mb-1.5">
                Student ID (Unique • တစ်ယောက်နှင့်တစ်ယောက် တူ၍မရပါ)
              </label>
              <input
                id="studentId"
                type="text"
                bind:value={authStudentId}
                required
                placeholder="TNT-XXXX"
                class="w-full bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-white text-sm font-mono uppercase outline-none focus:border-purple-500"
              />
            </div>
          {/if}

          <div>
            <label for="authEmail" class="block text-xs font-bold uppercase text-slate-400 mb-1.5">
              Email Address
            </label>
            <input
              id="authEmail"
              type="email"
              bind:value={authEmail}
              required
              placeholder="student@uit.edu.mm"
              class="w-full bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-white text-sm outline-none focus:border-purple-500"
            />
          </div>

          <div>
            <label for="authPass" class="block text-xs font-bold uppercase text-slate-400 mb-1.5">
              Password
            </label>
            <input
              id="authPass"
              type="password"
              bind:value={authPassword}
              required
              minlength="6"
              placeholder="••••••••"
              class="w-full bg-slate-950 border border-slate-800 rounded-xl px-4 py-3 text-white text-sm outline-none focus:border-purple-500"
            />
          </div>

          <button
            type="submit"
            disabled={loadingAction}
            class="w-full bg-linear-to-r from-purple-600 to-violet-600 text-white font-extrabold py-3.5 rounded-xl cursor-pointer mt-2"
          >
            {loadingAction ? 'Please wait...' : isLoginMode ? 'Sign In' : 'Create Account'}
          </button>
        </form>

        <div class="mt-6 text-center text-xs text-slate-400">
          {#if isLoginMode}
            အကောင့်မရှိသေးဘူးလား?
            <button
              type="button"
              onclick={() => {
                errorMessage = '';
                successMessage = '';
                isLoginMode = false;
              }}
              class="text-purple-400 font-bold hover:underline ml-1 cursor-pointer"
            >
              Sign Up အသစ်လုပ်မည်
            </button>
          {:else}
            အကောင့်ရှိပြီးသားလား?
            <button
              type="button"
              onclick={() => {
                errorMessage = '';
                successMessage = '';
                isLoginMode = true;
              }}
              class="text-purple-400 font-bold hover:underline ml-1 cursor-pointer"
            >
              Sign In ဝင်မည်
            </button>
          {/if}
        </div>
      </div>
    </div>
  {/if}

</div>